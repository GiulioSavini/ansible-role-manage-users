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
