# Design: generazione e rotazione chiavi SSH in `ansible-role-manage-users`

Data: 2026-06-26
Stato: approvato (design)

## Obiettivo

Estendere il ruolo `ansible-role-manage-users` per coprire scenari ricorrenti di
provisioning accessi su macchine Linux:

- creare utenti e relative coppie di chiavi SSH (privata + pubblica),
  eventualmente con **passphrase**;
- autorizzare l'accesso a chiave (gestione `authorized_keys` con permessi
  corretti su `.ssh/` e `authorized_keys`);
- **ruotare/sostituire** una chiave pubblica già presente con una nuova, in modo
  mirato (senza toccare le altre chiavi autorizzate);
- creare utenti con **sudo senza password** (già supportato, resta invariato).

Il ruolo resta pilotato da un'unica lista `manage_users_list`, comoda da esporre
in un survey AWX.

## Decisioni di design (approvate)

1. **Estensione del ruolo esistente**, non una repo nuova: niente duplicazione,
   tutto continua a passare da `manage_users_list`.
2. **Generazione chiavi sul controller Ansible**: le coppie nascono sulla
   macchina che lancia Ansible; le private restano centralizzate in una cartella
   locale, sul target viene distribuita solo la pubblica.
3. **Rotazione esplicita vecchia→nuova**: si indica la pub da rimuovere e la
   nuova da aggiungere; rimozione mirata che non tocca le altre chiavi.
4. Tipo di chiave di default: **ed25519**.

## Campi nuovi per ogni elemento di `manage_users_list`

| Campo | Tipo | Default | Descrizione |
|---|---|---|---|
| `generate_key` | bool | `false` | Genera una coppia di chiavi sul controller per questo utente |
| `key_type` | string | `ed25519` | `ed25519` oppure `rsa` |
| `key_bits` | int | `4096` | Dimensione chiave, usata solo per `rsa` |
| `key_passphrase` | string | *(nessuna)* | Passphrase della chiave privata (gestita con `no_log`) |
| `key_comment` | string | `<name>` | Commento della chiave pubblica |
| `authorize_generated_key` | bool | `true` | Aggiunge la pub generata agli `authorized_keys` dell'utente sul target |
| `ssh_keys_absent` | list | `[]` | Chiavi pubbliche da rimuovere dagli `authorized_keys` (rotazione mirata) |

Campi esistenti invariati: `name`, `ssh_keys`, `groups`, `append`, `shell`,
`create_home`, `sudo`, `sudo_nopasswd`, `state`, `remove`.

## Default globali nuovi (`defaults/main.yml`)

```yaml
manage_users_keys_dir: "{{ playbook_dir }}/keys"   # dove salvare le private sul controller
manage_users_default_key_type: ed25519
manage_users_default_key_bits: 4096
```

Layout su disco (controller): `keys/<utente>/id_<tipo>` (mode `0600`) e
`keys/<utente>/id_<tipo>.pub`.

## Flusso dei task

1. **Validazione input** (esistente) — `name` obbligatorio.
2. **Genera chiavi** (`tasks/generate_keys.yml`, nuovo) — modulo
   `community.crypto.openssh_keypair`, `delegate_to: localhost`,
   `run_once: true`, `become: false`. Crea la coppia in `manage_users_keys_dir`,
   con `passphrase` se indicata, `mode: '0600'` sulla privata. Idempotente: se la
   chiave esiste non viene rigenerata. `no_log: true` sul task per non esporre la
   passphrase. Solo per gli utenti con `generate_key: true` e
   `state == present`.
3. **Crea/rimuovi utenti** (esistente) — invariato.
4. **Autorizza chiavi** (esistente, esteso) — `ansible.posix.authorized_key`
   `state=present`, `exclusive=false` per le `ssh_keys` elencate; in più
   aggiunge la pubblica generata quando `authorize_generated_key` è vero (lettura
   del contenuto del file `.pub` dal controller). Permessi su `.ssh/` (700) e
   `authorized_keys` (600) gestiti dal modulo (`manage_dir` di default).
5. **Rotazione mirata** (nuovo) — `ansible.posix.authorized_key` `state=absent`
   in loop su `ssh_keys_absent`: rimuove solo le pub elencate.
6. **Sudo NOPASSWD** (esistente) — drop-in in `/etc/sudoers.d/<name>` con
   `validate: visudo`, invariato.
7. **Cleanup sudoers** (esistente) — rimozione drop-in quando utente assente o
   sudo disabilitato, invariato.

L'ordine garantisce che la pub sia generata prima di essere autorizzata, e che la
rotazione (rimozione vecchia) avvenga insieme all'aggiunta della nuova.

## Sicurezza

- `requirements.yml`: aggiungere `community.crypto` (oltre a `ansible.posix`).
- `.gitignore`: ignorare `keys/` — le chiavi private non devono mai finire su git.
- `no_log: true` sui task che maneggiano `key_passphrase`.
- README: raccomandare `ansible-vault` o storage out-of-band per le private e per
  le passphrase; ricordare che la generazione sul controller centralizza le
  private in `manage_users_keys_dir`.

## Test

La modifica va testata (regola di progetto). Test minimo, leggero, senza
Molecule/Docker:

- `ansible-playbook --syntax-check` sul playbook di test;
- `ansible-lint` sul ruolo;
- esecuzione in **check mode** contro `localhost` con un utente di esempio
  (`tests/test.yml`).

Un CI completo con Molecule potrà essere aggiunto in un secondo momento.

## Esempio d'uso

```yaml
- name: Setup utenti e chiavi
  hosts: all
  become: true
  roles:
    - role: ansible-role-manage-users
      vars:
        manage_users_list:
          # nuovo utente con chiave generata + passphrase + sudo nopasswd
          - name: deploy
            generate_key: true
            key_type: ed25519
            key_passphrase: "{{ vault_deploy_passphrase }}"
            groups: [wheel]            # su Debian/Ubuntu: [sudo]
            sudo: true
            sudo_nopasswd: true

          # rotazione: rimuove la vecchia pub, autorizza la nuova
          - name: alice
            ssh_keys:
              - "ssh-ed25519 AAAA...NUOVA alice@new"
            ssh_keys_absent:
              - "ssh-ed25519 AAAA...VECCHIA alice@old"
```

## Fuori scope (per ora)

- CI con Molecule/Docker.
- Generazione chiavi sul target (scelto: solo sul controller).
- Modalità `exclusive` per gli `authorized_keys` (scelto: rotazione mirata).
