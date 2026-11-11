---
date: 2026-11-10
categories:
  - Linux
tags:
  - hyprland
  - systemd
  - energia
  - caelestia
---

# Como o Caelestia-shell hibernava meu PC sozinho (e como eu desabilitei)

Meu PC estava **travando a tela sozinho** depois de 3 minutos de ociosidade e,
pior, **suspendendo e depois hibernando** aos 10 minutos. Como não existia
nenhuma opção visível nas configurações do sistema, fui investigar a fundo — e
a resposta foi bem menos óbvia do que eu imaginava.

<!-- more -->

## O sintoma

- A tela **trava sozinha** após ~3 min sem mexer no mouse/teclado.
- Aos ~10 min, a máquina **suspende** e, duas horas depois, tenta **hibernar**.
- Nas "Configurações do Sistema" **não existe** nenhuma opção de suspensão,
  hibernação ou tempo de ociosidade.

O detalhe importante: eu uso **Hyprland** + **Caelestia-shell** (Quickshell),
e **não** havia `gnome-control-center` nem `plasma-workspace` instalados. Ou
seja, o painel de "Energia" do GNOME/KDE simplesmente **não existe** nessa
máquina. Por isso "não tinha opção visível": eu estava procurando no lugar
errado — o controle de energia aqui é 100% do **systemd** e do **shell**.

## O que eu suspeitei primeiro (e estava errado)

Meu primeiro palpite foi um daemon de ociosidade do Hyprland. Verifiquei:

```bash
pacman -Qq | grep -Ei 'hypridle|swayidle|hyprland-plugin'   # vazio
ls ~/.local/share/hyprland/plugins/                          # inexistente
systemctl --user list-timers --all                           # nenhum timer
ls ~/.config/autostart/                                     # vazio
```

Nada. Não há `hypridle`, não há `swayidle`, não há plugin do Hyprland
carregado, não há serviço de usuário de energia, não há autostart. Os únicos
daemons vivos eram `power-profiles-daemon` e `upowerd`, e nenhum dos dois
suspende ou hiberna.

## A causa real: defaults compilados no Caelestia-shell

O culpado era o próprio shell. No pacote `caelestia-shell` (2.5.0) os
**timeouts de ociosidade são defaults compilados em C++**, em
`plugin/src/Caelestia/Config/generalconfig.hpp`:

```cpp
CONFIG_LIST(GeneralIdleTimeoutList, timeouts,
    DEFAULT_ARG({
        vmap({ { u"timeout"_s, 180 }, { u"idleAction"_s, u"lock"_s } }),
        vmap({ { u"timeout"_s, 300 },
               { u"idleAction"_s, u"dpms off"_s },
               { u"returnAction"_s, u"dpms on"_s } }),
        vmap({ { u"timeout"_s, 600 },
               { u"idleAction"_s, QStringList{ u"suspendThenHibernate"_s } } }),
    }))
```

Traduzido:

| Ociosidade | Ação | Efeito |
|---|---|---|
| **3 min** | `lock` | **trava a tela** |
| 5 min | `dpms off` / `dpms on` | apaga a tela (normal) |
| **10 min** | `suspendThenHibernate` | **suspende e depois hiberna** |

Mais dois defaults relacionados:

```cpp
CONFIG_PROPERTY(bool, lockBeforeSleep, true)   // trava a tela ao suspender
CONFIG_PROPERTY(int, criticalLevel, 3)          // hiberna com 3% de bateria
```

Como esses valores são **defaults do código compilado** (e não do arquivo de
configuração do usuário), eles **não aparecem** em lugar nenhum da interface.
O arquivo `~/.config/caelestia/shell.json` só sobrescreve o que você escrever
explicitamente nele.

### Como a ação vira uma chamada de sistema

O `IdleMonitors.qml` do shell traduz a ação para a API do systemd:

```qml
// modules/IdleMonitors.qml
else if (!SessionManager.exec(action))
    Quickshell.execDetached(action);
```

E o `SessionManager` (em `libcaelestia-services.so`) mapeia o nome para a
chamada D-Bus do `systemd-logind`:

```cpp
{ u"suspendthenhibernate"_s, &SessionManager::suspendThenHibernate },
```

Ou seja: `suspendThenHibernate` vira
`org.freedesktop.login1.Manager.SuspendThenHibernate` — o logind suspende e
agenda a hibernação. E o **logind**, por padrão, usa
`SleepOperation=suspend-then-hibernate`, com `HibernateDelaySec=2h`
(ou o alarme ACPI de bateria baixa, se a máquina tiver bateria).

## Antes de mexer: a hibernação sequer funcionava aqui

Um detalhe importante que descobri no caminho: **esta máquina não conseguia
hibernar de verdade**.

```bash
$ cat /sys/power/state
freeze mem disk          # o kernel suporta
$ swapon --show
NAME       TYPE       SIZE USED PRIO
/dev/zram0 partition     4G   0B  100   # só zram!
$ free -h
Mem.:  15Gi
$ cat /proc/cmdline
root=PARTUUID=257df0ca-... rw rootfstype=ext4   # sem resume=
```

- A imagem de hibernação (`/sys/power/image_size` = 6,5 GB) precisa de swap
  **em disco**; aqui só existe `zram0` de 4 GB, que é volátil.
- Não há `resume=` no cmdline, nem partição/swapfile de hibernação.

Ou seja, o "suspend-then-hibernate" só ia dar trabalho: o sistema passaria 2 h
suspenso para depois falhar na hora de hibernar (`Failed to find location to
hibernate to`). Além disso, a máquina é um **desktop**
(`/sys/class/power_supply/` vazio, `chassis_type=3`), então nem o alarme de
bateria baixa se aplicava.

## A solução

### 1. Sobrescrever os timeouts no `shell.json`

O `shell.json` é recarregado automaticamente pelo shell (ele usa um
`QFileSystemWatcher`), então não precisa reiniciar nada:

```json
{
    "general": {
        "idle": {
            "lockBeforeSleep": false,
            "timeouts": [
                {
                    "timeout": 300,
                    "idleAction": "dpms off",
                    "returnAction": "dpms on"
                }
            ]
        },
        "battery": {
            "criticalLevel": 0
        }
    }
}
```

O que mudou:

- `"timeouts"` **substitui** a lista inteira (não é merge): tirei a entrada de
  `lock` (3 min) e a de `suspendThenHibernate` (10 min), e mantive só o
  desligamento de tela (5 min), que é comportamento normal de desktop.
- `"lockBeforeSleep": false` → a tela não trava mais quando o sistema suspende.
- `"criticalLevel": 0` → desliga a hibernação por bateria crítica.

### 2. Tirar o `suspend-then-hibernate` do atalho do Hyprland

O atalho `SUPER + SHIFT + L` executava o comando direto do systemd:

```lua
-- ~/.config/hypr/variables.lua
sleepGestureCmd = "systemctl suspend-then-hibernate",
```

Como este arquivo vem dos dotfiles do Caelestia, ele existe em dois lugares —
e **os dois** precisam mudar, senão o `caelestia update` reverte:

```lua
sleepGestureCmd = "systemctl suspend",
```

| Arquivo | Por quê |
|---|---|
| `~/.config/hypr/variables.lua` | é o que o Hyprland lê |
| `~/.local/state/caelestia/dots/hypr/variables.lua` | é a cópia que o `caelestia update` re-aplica |

Depois é só recarregar:

```bash
hyprctl reload
```

### 3. (Opcional, exige root) Blindar o systemd

O logind continua com `SleepOperation=suspend-then-hibernate` (padrão do
systemd). Nada mais dispara isso automaticamente, mas dá pra travar a porta:

```bash
sudo mkdir -p /etc/systemd/logind.conf.d
printf '[Login]\nSleepOperation=suspend\n' | \
  sudo tee /etc/systemd/logind.conf.d/10-no-hibernate.conf
sudo systemctl kill -s HUP systemd-logind
```

Use `kill -s HUP` em vez de `restart`: reiniciar o `systemd-logind` derruba a
sessão.

## Como conferir se funcionou

```bash
# o logind ainda oferece hibernação?
busctl get-property org.freedesktop.login1 /org/freedesktop/login1 \
  org.freedesktop.login1.Manager CanHibernate

# qual o modo de suspensão em uso?
busctl get-property org.freedesktop.login1 /org/freedesktop/login1 \
  org.freedesktop.login1.Manager SleepOperation
```

E o teste definitivo: deixar o PC ocioso por 5+ minutos. A **tela deve
apagar**, mas **não deve travar nem suspender**.

Para desfazer tudo: basta apagar o bloco `"general"` do `shell.json` (volta
aos defaults do shell) e restaurar a linha `suspend-then-hibernate` no
`variables.lua`.

## O que aprendi

1. **"Não existe opção na interface" ≠ "não existe opção".** Aqui a opção
   existia, estava compilada dentro de um binário. Quando um comportamento
   parece mágico no Wayland, o culpado costuma estar num daemon de usuário
   ou em defaults de um shell — não no compositor.
2. **Defaults de pacotes são documentação invisível.** Abrir o fonte do
   pacote (`pacman -Ql`, tarball do AUR, `strings` no `.so`) responde em
   minutos o que a busca na interface não responde.
3. **Nem toda configuração mora no seu `$HOME`.** São várias camadas:
   kernel → systemd → shell → dotfiles. Diagnosticar exige saber em qual
   camada a opção está.
4. **Antes de "desabilitar", verifique se a coisa funcionava.** A hibernação
   jamais teria funcionado aqui (zram volátil, sem `resume=`). Se eu tivesse
   só desabilitado, teria deixado um timer de 2 h quebrado no caminho.

## Referências

- `man systemd-sleep.conf` — `HibernateDelaySec`, `AllowHibernation`
- `man systemd-logind.conf` — `SleepOperation`, `HandleHibernateKey`
- `pacman -Ql caelestia-shell` — lista os QMLs do shell
- `strings /usr/lib/qt6/qml/Caelestia/lib/libcaelestia-services.so` — revela
  os nomes das ações do `SessionManager`
