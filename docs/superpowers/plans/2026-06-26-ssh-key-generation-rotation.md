# SSH Key Generation & Rotation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Estendere `ansible-role-manage-users` per generare coppie di chiavi SSH sul controller (con passphrase opzionale), autorizzarle sui target e ruotare/sostituire chiavi pubbliche in modo mirato.

**Architecture:** Si aggiungono campi per-utente a `manage_users_list` e nuovi task al ruolo. La generazione chiavi gira sul controller (`delegate_to: localhost`, `run_once: true`); la pub generata viene letta con `lookup('file', ...)` e autorizzata sui target via `ansible.posix.authorized_key`; la rotazione rimuove le pub elencate in `ssh_keys_absent`.

**Tech Stack:** Ansible core 2.16, `community.crypto.openssh_keypair`, `ansible.posix.authorized_key`. Test con `ansible-playbook --syntax-check` e run reale di sola generazione su `localhost`.

---

## File Structure

- Modify `requirements.yml` — aggiunge `community.crypto`.
- Modify `meta/main.yml` — aggiunge `community.crypto` alla lista `collections`.
- Modify `defaults/main.yml` — nuovi default globali + commenti sui campi nuovi.
- Modify `.gitignore` — ignora `keys/`.
- Create `tasks/generate_keys.yml` — directory chiavi + `openssh_keypair` sul controller.
- Modify `tasks/main.yml` — include keygen, autorizza pub generata, task rotazione `ssh_keys_absent`.
- Create `tests/test.yml` — playbook di test (generazione su `localhost`).
- Modify `README.md` — documenta i campi nuovi e un esempio.

---

## Task 1: Dipendenze, default e .gitignore

**Files:**
- Modify: `requirements.yml`
- Modify: `meta/main.yml`
- Modify: `defaults/main.yml`
- Modify: `.gitignore`

- [ ] **Step 1: Aggiungi `community.crypto` a `requirements.yml`**

Sostituisci l'intero contenuto di `requirements.yml` con:

```yaml
---
collections:
  - name: ansible.posix
  - name: community.crypto
```

- [ ] **Step 2: Aggiungi `community.crypto` a `meta/main.yml`**

In `meta/main.yml`, sostituisci il blocco:

```yaml
collections:
  - ansible.posix
```

con:

```yaml
collections:
  - ansible.posix
  - community.crypto
```

- [ ] **Step 3: Aggiungi i default globali in `defaults/main.yml`**

In fondo a `defaults/main.yml`, dopo la riga `manage_users_sudoers_dir: /etc/sudoers.d`, aggiungi:

```yaml

# --- Generazione/rotazione chiavi SSH ---
# Directory sul CONTROLLER dove salvare le coppie generate: keys/<utente>/id_<tipo>
manage_users_keys_dir: "{{ playbook_dir }}/keys"
# Tipo di chiave di default per generate_key
manage_users_default_key_type: ed25519
# Dimensione chiave (solo per type=rsa)
manage_users_default_key_bits: 4096
```

E nel blocco di commenti iniziale che elenca i campi, dopo la riga `#   remove ...`, aggiungi:

```yaml
#   generate_key            (optional) genera una coppia di chiavi sul controller; default false
#   key_type                (optional) ed25519|rsa; default ed25519
#   key_bits                (optional) dimensione chiave per rsa; default 4096
#   key_passphrase          (optional) passphrase della chiave privata (no_log)
#   key_comment             (optional) commento della pubblica; default <name>
#   authorize_generated_key (optional) autorizza la pub generata sul target; default true
#   ssh_keys_absent         (optional) lista di pub da rimuovere dagli authorized_keys (rotazione)
```

- [ ] **Step 4: Ignora `keys/` in `.gitignore`**

Aggiungi in fondo a `.gitignore`:

```gitignore

# Chiavi private generate dal ruolo: non committarle mai
keys/
```

- [ ] **Step 5: Verifica che le collection necessarie siano installate**

Run: `ansible-galaxy collection list 2>/dev/null | grep -Ei 'posix|crypto'`
Expected: righe per `ansible.posix` e `community.crypto` (entrambe presenti).

- [ ] **Step 6: Commit**

```bash
git add requirements.yml meta/main.yml defaults/main.yml .gitignore
git commit -m "feat: add defaults and dependency for SSH key generation/rotation"
```

---

## Task 2: Test harness (test che fallisce)

**Files:**
- Create: `tests/test.yml`

- [ ] **Step 1: Scrivi il playbook di test**

Crea `tests/test.yml`:

```yaml
---
# Test locale: genera una coppia di chiavi sul controller (nessun utente OS creato).
# Esegue solo i task di keygen tramite il tag 'manage-users-keygen'.
- name: Test generazione chiavi SSH (solo controller)
  hosts: localhost
  connection: local
  gather_facts: false
  vars:
    manage_users_keys_dir: /tmp/manage-users-test-keys
    manage_users_list:
      - name: testkeygen
        generate_key: true
        key_type: ed25519
        key_comment: "testkeygen@ci"
  roles:
    - role: ansible-role-manage-users
```

- [ ] **Step 2: Esegui il syntax-check (deve passare anche ora)**

Run: `ansible-playbook tests/test.yml --syntax-check`
Expected: `playbook: tests/test.yml` senza errori.

- [ ] **Step 3: Esegui il test di generazione (deve FALLIRE ora)**

Run:
```bash
rm -rf /tmp/manage-users-test-keys
ANSIBLE_ROLES_PATH=.. ansible-playbook tests/test.yml --tags manage-users-keygen
test -f /tmp/manage-users-test-keys/testkeygen/id_ed25519 && echo KEYGEN-OK || echo KEYGEN-FAIL
```
Expected: NON stampa `KEYGEN-OK` (il tag `manage-users-keygen` non esiste ancora / nessuna chiave creata) → `KEYGEN-FAIL`.

- [ ] **Step 4: Commit**

```bash
git add tests/test.yml
git commit -m "test: add local SSH keygen test playbook"
```

---

## Task 3: Generazione chiavi sul controller

**Files:**
- Create: `tasks/generate_keys.yml`
- Modify: `tasks/main.yml`

- [ ] **Step 1: Crea `tasks/generate_keys.yml`**

```yaml
---
- name: Crea le directory delle chiavi sul controller
  ansible.builtin.file:
    path: "{{ manage_users_keys_dir }}/{{ item.name }}"
    state: directory
    mode: '0700'
  loop: "{{ manage_users_list }}"
  loop_control:
    label: "{{ item.name }}"
  when:
    - (item.state | default(manage_users_default_state)) == 'present'
    - (item.generate_key | default(false)) | bool
  delegate_to: localhost
  run_once: true
  become: false
  tags:
    - manage-users
    - manage-users-keygen

- name: Genera le coppie di chiavi SSH sul controller
  community.crypto.openssh_keypair:
    path: "{{ manage_users_keys_dir }}/{{ item.name }}/id_{{ item.key_type | default(manage_users_default_key_type) }}"
    type: "{{ item.key_type | default(manage_users_default_key_type) }}"
    size: "{{ (item.key_bits | default(manage_users_default_key_bits)) if (item.key_type | default(manage_users_default_key_type)) == 'rsa' else omit }}"
    passphrase: "{{ item.key_passphrase | default(omit) }}"
    comment: "{{ item.key_comment | default(item.name) }}"
    mode: '0600'
  loop: "{{ manage_users_list }}"
  loop_control:
    label: "{{ item.name }}"
  when:
    - (item.state | default(manage_users_default_state)) == 'present'
    - (item.generate_key | default(false)) | bool
  delegate_to: localhost
  run_once: true
  become: false
  no_log: true
  tags:
    - manage-users
    - manage-users-keygen
```

- [ ] **Step 2: Includi il keygen all'inizio di `tasks/main.yml`**

In `tasks/main.yml`, subito dopo il task `Valida input` (prima di `Crea o rimuovi utenti`), inserisci:

```yaml
- name: Genera chiavi SSH sul controller (se richiesto)
  ansible.builtin.include_tasks: generate_keys.yml
  when: manage_users_list | selectattr('generate_key', 'defined') | selectattr('generate_key') | list | length > 0
  tags:
    - manage-users
    - manage-users-keygen
```

- [ ] **Step 3: Esegui il syntax-check**

Run: `ansible-playbook tests/test.yml --syntax-check`
Expected: nessun errore.

- [ ] **Step 4: Esegui il test di generazione (ora deve PASSARE)**

Run:
```bash
rm -rf /tmp/manage-users-test-keys
ANSIBLE_ROLES_PATH=.. ansible-playbook tests/test.yml --tags manage-users-keygen
test -f /tmp/manage-users-test-keys/testkeygen/id_ed25519 && echo KEYGEN-OK || echo KEYGEN-FAIL
test -f /tmp/manage-users-test-keys/testkeygen/id_ed25519.pub && echo PUB-OK || echo PUB-FAIL
stat -c '%a' /tmp/manage-users-test-keys/testkeygen/id_ed25519
```
Expected: `KEYGEN-OK`, `PUB-OK`, e permessi `600` sulla chiave privata.

- [ ] **Step 5: Verifica l'idempotenza (seconda run = changed=0 sul keygen)**

Run:
```bash
ANSIBLE_ROLES_PATH=.. ansible-playbook tests/test.yml --tags manage-users-keygen | tail -3
```
Expected: nel recap `changed=0` (la chiave esiste già, non viene rigenerata).

- [ ] **Step 6: Pulisci e committa**

```bash
rm -rf /tmp/manage-users-test-keys
git add tasks/generate_keys.yml tasks/main.yml
git commit -m "feat: generate SSH keypairs on the controller"
```

---

## Task 4: Autorizza la pub generata sul target

**Files:**
- Modify: `tasks/main.yml`

- [ ] **Step 1: Aggiungi il task di autorizzazione della pub generata**

In `tasks/main.yml`, subito DOPO il task esistente `Configura chiavi SSH autorizzate`, aggiungi:

```yaml
- name: Autorizza la chiave generata sul controller
  ansible.posix.authorized_key:
    user: "{{ item.name }}"
    key: "{{ lookup('file', manage_users_keys_dir + '/' + item.name + '/id_' + (item.key_type | default(manage_users_default_key_type)) + '.pub') }}"
    state: present
    exclusive: false
  loop: "{{ manage_users_list }}"
  loop_control:
    label: "{{ item.name }}"
  when:
    - (item.state | default(manage_users_default_state)) == 'present'
    - (item.generate_key | default(false)) | bool
    - (item.authorize_generated_key | default(true)) | bool
  tags:
    - manage-users
    - manage-users-ssh
```

- [ ] **Step 2: Syntax-check**

Run: `ansible-playbook tests/test.yml --syntax-check`
Expected: nessun errore.

- [ ] **Step 3: Verifica check-mode su localhost (no modifiche reali)**

Run:
```bash
rm -rf /tmp/manage-users-test-keys
ANSIBLE_ROLES_PATH=.. ansible-playbook tests/test.yml --tags manage-users-keygen
ANSIBLE_ROLES_PATH=.. ansible-playbook tests/test.yml --check --tags manage-users-ssh 2>&1 | tail -5
```
Expected: il play in check-mode non solleva errori di templating sul `lookup` della pub (la pub esiste dal run precedente). Eventuale fallimento nel modificare `authorized_keys` dell'utente locale è atteso/ignorabile in check-mode; conta che non ci siano errori di sintassi/variabili.

- [ ] **Step 4: Pulisci e committa**

```bash
rm -rf /tmp/manage-users-test-keys
git add tasks/main.yml
git commit -m "feat: authorize generated public key on targets"
```

---

## Task 5: Rotazione mirata (`ssh_keys_absent`)

**Files:**
- Modify: `tasks/main.yml`

- [ ] **Step 1: Aggiungi il task di rimozione chiavi**

In `tasks/main.yml`, subito DOPO il task `Autorizza la chiave generata sul controller`, aggiungi:

```yaml
- name: Rimuovi chiavi SSH ruotate (rotazione mirata)
  ansible.posix.authorized_key:
    user: "{{ item.0.name }}"
    key: "{{ item.1 }}"
    state: absent
  loop: "{{ manage_users_list | subelements('ssh_keys_absent', skip_missing=True) }}"
  loop_control:
    label: "{{ item.0.name }}"
  when: (item.0.state | default(manage_users_default_state)) == 'present'
  tags:
    - manage-users
    - manage-users-ssh
```

- [ ] **Step 2: Syntax-check**

Run: `ansible-playbook tests/test.yml --syntax-check`
Expected: nessun errore.

- [ ] **Step 3: Verifica che `subelements` con `ssh_keys_absent` assente non rompa**

Run:
```bash
ANSIBLE_ROLES_PATH=.. ansible-playbook tests/test.yml --check --tags manage-users-ssh 2>&1 | tail -5
```
Expected: nessun errore "subelements" (il `skip_missing=True` salta gli utenti senza `ssh_keys_absent`, come l'utente `testkeygen` del test).

- [ ] **Step 4: Commit**

```bash
git add tasks/main.yml
git commit -m "feat: targeted SSH key rotation via ssh_keys_absent"
```

---

## Task 6: Documentazione README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Aggiungi le righe alla tabella dei campi**

In `README.md`, nella tabella dei campi utente, dopo la riga di `remove`, aggiungi:

```markdown
| `generate_key` | bool    | `false`                            | Genera una coppia di chiavi sul **controller** (`keys/<name>/id_<type>`)   |
| `key_type`      | string  | `ed25519`                          | `ed25519` o `rsa`                                                          |
| `key_bits`      | int     | `4096`                             | Dimensione chiave (solo per `rsa`)                                         |
| `key_passphrase`| string  | *(nessuna)*                        | Passphrase della chiave privata (gestita con `no_log`)                    |
| `key_comment`   | string  | `<name>`                           | Commento della chiave pubblica                                            |
| `authorize_generated_key` | bool | `true`                  | Aggiunge la pub generata agli `authorized_keys` sul target                |
| `ssh_keys_absent`| list   | `[]`                               | Pub da rimuovere dagli `authorized_keys` (rotazione mirata)               |
```

- [ ] **Step 2: Aggiungi una sezione "Generazione e rotazione chiavi"**

Dopo l'esempio playbook esistente, aggiungi:

````markdown
## Generazione e rotazione chiavi SSH

Le chiavi vengono generate **sul controller** in `manage_users_keys_dir`
(default `keys/` accanto al playbook). Le private NON vanno committate: la
cartella `keys/` è in `.gitignore`. Conserva private e passphrase con
`ansible-vault` o storage out-of-band.

```yaml
- name: Setup utenti e chiavi
  hosts: all
  become: true
  roles:
    - role: ansible-role-manage-users
      vars:
        manage_users_list:
          # utente nuovo con chiave generata + passphrase + sudo nopasswd
          - name: deploy
            generate_key: true
            key_type: ed25519
            key_passphrase: "{{ vault_deploy_passphrase }}"
            groups: [wheel]          # su Debian/Ubuntu: [sudo]
            sudo: true
            sudo_nopasswd: true

          # rotazione: rimuove la vecchia pub, autorizza la nuova
          - name: alice
            ssh_keys:
              - "ssh-ed25519 AAAA...NUOVA alice@new"
            ssh_keys_absent:
              - "ssh-ed25519 AAAA...VECCHIA alice@old"
```

Tag utili: `--tags manage-users-keygen` (solo generazione),
`--tags manage-users-ssh` (solo chiavi autorizzate/rotazione).
````

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: document SSH key generation and rotation fields"
```

---

## Task 7: Verifica finale e push

- [ ] **Step 1: Syntax-check completo**

Run: `ansible-playbook tests/test.yml --syntax-check`
Expected: nessun errore.

- [ ] **Step 2: Test funzionale keygen end-to-end**

Run:
```bash
rm -rf /tmp/manage-users-test-keys
ANSIBLE_ROLES_PATH=.. ansible-playbook tests/test.yml --tags manage-users-keygen
test -f /tmp/manage-users-test-keys/testkeygen/id_ed25519 && stat -c '%a' /tmp/manage-users-test-keys/testkeygen/id_ed25519
rm -rf /tmp/manage-users-test-keys
```
Expected: file presente, permessi `600`.

- [ ] **Step 3: Push del branch**

```bash
git push -u origin feature/ssh-key-generation-rotation
```

- [ ] **Step 4: (Opzionale) Apri la PR**

```bash
gh pr create --fill --title "feat: SSH key generation and rotation" \
  --body "Aggiunge generazione chiavi sul controller (con passphrase), autorizzazione della pub generata e rotazione mirata via ssh_keys_absent. Vedi docs/superpowers/specs e plans."
```

---

## Self-Review Notes

- **Spec coverage:** generazione chiavi (Task 3), passphrase (`key_passphrase`, Task 3), autorizzazione pub generata (Task 4), rotazione esplicita vecchia→nuova (`ssh_keys_absent`, Task 5), permessi `.ssh`/`authorized_keys` (gestiti da `authorized_key`), sudo NOPASSWD (esistente, invariato), sicurezza private (`.gitignore` + `no_log`, Task 1/3), test leggero (Task 2/7), dipendenza `community.crypto` (Task 1). Tutti i requisiti dello spec hanno un task.
- **Tipi/nomi coerenti:** `manage_users_keys_dir`, `manage_users_default_key_type`, `manage_users_default_key_bits` usati identici in defaults, generate_keys.yml e main.yml. Tag `manage-users-keygen` / `manage-users-ssh` coerenti tra task e comandi di test.
- **Nota di test:** l'autorizzazione sul target e la rotazione richiedono un host gestito reale per un test funzionale completo; qui sono verificate con syntax-check + check-mode. Un CI Molecule è esplicitamente fuori scope (vedi spec).
