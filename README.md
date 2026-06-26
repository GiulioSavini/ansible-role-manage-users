# ansible-role-manage-users

Ruolo Ansible per gestire utenti Linux in modo idempotente: creazione/rimozione,
chiavi SSH autorizzate, appartenenza a gruppi e sudo senza password tramite
`/etc/sudoers.d/`. Pensato per essere comodo con **AWX** (basta esporre
`manage_users_list` in un survey).

## Sistemi supportati

Tutti i sistemi Linux in cui `ansible.builtin.user` funziona: Debian/Ubuntu,
RHEL/CentOS/Rocky/Alma/Oracle Linux, Fedora, openSUSE/SLES, ecc.

## Requisiti

- Ansible >= 2.12
- Collection `ansible.posix` (installabile con `ansible-galaxy collection install -r requirements.yml`)

## Variabili

Tutto si pilota da un'unica lista, `manage_users_list`. Ogni elemento descrive
un utente:

| Campo          | Tipo    | Default                            | Descrizione                                              |
|----------------|---------|------------------------------------|----------------------------------------------------------|
| `name`         | string  | *(obbligatorio)*                   | Nome dell'utente                                         |
| `ssh_keys`     | list    | `[]`                               | Chiavi pubbliche SSH da autorizzare                      |
| `groups`       | list    | *(nessuno)*                        | Gruppi supplementari (es. `[wheel]` su RHEL, `[sudo]` su Debian) |
| `append`       | bool    | `true`                             | `true` = aggiunge ai groups esistenti, `false` = sostituisce |
| `shell`        | string  | `/bin/bash`                        | Shell di login                                           |
| `create_home`  | bool    | `true`                             | Crea la home directory                                   |
| `sudo`         | bool    | `true`                             | Abilita sudo via `/etc/sudoers.d/<name>`                 |
| `sudo_nopasswd`| bool    | `true`                             | Rende il sudo senza password                             |
| `state`        | string  | `present`                          | `present` o `absent`                                     |
| `remove`       | bool    | `false`                            | Con `state=absent`, elimina anche la home                |
| `generate_key` | bool    | `false`                            | Genera una coppia di chiavi sul **controller** (`keys/<name>/id_<type>`)   |
| `key_type`      | string  | `ed25519`                          | `ed25519` o `rsa`                                                          |
| `key_bits`      | int     | `4096`                             | Dimensione chiave (solo per `rsa`)                                         |
| `key_passphrase`| string  | *(nessuna)*                        | Passphrase della chiave privata (gestita con `no_log`)                    |
| `key_comment`   | string  | `<name>`                           | Commento della chiave pubblica                                            |
| `authorize_generated_key` | bool | `true`                  | Aggiunge la pub generata agli `authorized_keys` sul target                |
| `ssh_keys_absent`| list   | `[]`                               | Pub da rimuovere dagli `authorized_keys` (rotazione mirata)               |

I default globali (se vuoi cambiarli) sono in `defaults/main.yml`
(`manage_users_default_*`).

## Esempio playbook

```yaml
---
- name: Setup utenti
  hosts: all
  become: true
  gather_facts: false
  roles:
    - role: ansible-role-manage-users
      vars:
        manage_users_list:
          - name: xxxxxxxx
            ssh_keys:
              - "ssh-rsa xxxxxxxxxxttttt"
            groups: [wheel]       # su Debian/Ubuntu usa [sudo]
            sudo: true
            sudo_nopasswd: true

          - name: alice
            ssh_keys:
              - "ssh-ed25519 AAAAC3... alice@laptop"
            groups: [docker]
            sudo: false            # niente sudoers per lei

          - name: vecchio_utente
            state: absent
            remove: true           # rimuove anche la home
```

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

## Uso con AWX

1. In Job Template, collega il ruolo nel playbook.
2. Crea un **Survey** con una variabile `manage_users_list` di tipo
   *Textarea* e formato **YAML**, con default simile a:

   ```yaml
   - name: xxxxxxx
     ssh_keys:
       - "ssh-rsa AAAA..."
     groups: [wheel]
   ```

3. Lancia il job sull'inventory target.

## Tag disponibili

- `manage-users` — tutti i task del ruolo
- `manage-users-account` — solo creazione/rimozione utente
- `manage-users-ssh` — solo chiavi SSH
- `manage-users-sudo` — solo sudoers

## Licenza

MIT
