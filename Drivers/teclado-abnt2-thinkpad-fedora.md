# Corrigindo a tecla `/ ?` (ABNT2) que não funciona em ThinkPad no Fedora

> Teclado ABNT2 de um Lenovo ThinkPad no Fedora Workstation 44 (GNOME): a tecla `/ ?` não digitava nada. Este guia mostra como descobrir o código que a tecla envia e remapeá-la com o **hwdb** do udev.

## Sintoma

Após instalar o Fedora, a tecla `/ ?` (ao lado do Shift direito) não produzia `/` nem `?`.

**Quebra-galho enquanto não corrige** (ABNT2 com layout Português Brasil):

- `AltGr + Q` → `/`
- `AltGr + W` → `?`

## Ambiente

- Lenovo ThinkPad com teclado ABNT2
- Fedora Linux 44 (Workstation Edition)
- Layout de teclado: **Português (Brasil)**

## Diagnóstico

Primeiro, confira em **Configurações → Teclado → Fontes de entrada** se o layout é **Português (Brasil)**. Se estiver em "Inglês (EUA)", basta trocar.

Se o layout estiver certo, descubra qual código a tecla está enviando:

```bash
sudo showkey --scancodes
```

Aperte a tecla `/ ?` e aguarde cerca de 10 segundos até o comando encerrar sozinho.

No meu caso a saída foi:

```
0xe0 0x1d 0xe0 0x9d
```

Esse é o par pressionar/soltar do **Ctrl direito**, tecla que esse notebook nem possui. Ou seja, a tecla `/ ?` estava enviando o código errado, e o sistema achava que eu apertava Ctrl.

> **Observação:** o `showkey` pode exibir "modo kb era DESCONHECIDO" em terminais gráficos (GNOME/Wayland). Se ele não mostrar nada ao apertar a tecla, use o `evtest`:
>
> ```bash
> sudo dnf install evtest
> sudo evtest
> ```
>
> Escolha o dispositivo do teclado (algo como "AT Translated Set 2 keyboard"), aperte a tecla e observe o valor de `MSC_SCAN`.
>
> Se nenhum evento aparecer, o sistema não recebe o sinal da tecla, e o problema provavelmente é físico (teclado ou flat cable).

## Como o Linux processa uma tecla

1. Você aperta a tecla e o teclado envia um **scancode** (um número) ao computador.
2. O **kernel** converte o scancode em um **keycode** (por exemplo, "Ctrl direito" ou "barra").
3. O **layout** (Português Brasil) converte o keycode no caractere final: `/`, `?` etc.

O problema estava na etapa 1: o scancode enviado era interpretado como Ctrl direito. A solução é dizer ao sistema: "quando chegar este scancode, trate como a tecla `ro`".

`ro` é o nome interno da tecla extra do ABNT2, a `/ ?` ao lado do Shift direito.

## Solução

### 1. Converter o scancode para o formato do hwdb

O `showkey` mostrou `0xe0 0x1d`. O prefixo `e0` indica uma tecla "estendida", e o kernel representa isso somando `0x80` ao segundo byte:

```
0x1d + 0x80 = 0x9d
```

Portanto, o código que usaremos no hwdb é `9d`.

### 2. Criar o arquivo de regra

```bash
sudo nano /etc/udev/hwdb.d/90-teclado.hwdb
```

Conteúdo (a segunda linha **precisa começar com um espaço**):

```
evdev:atkbd:dmi:bvn*:bvr*:bd*:svnLENOVO*:pn*
 KEYBOARD_KEY_9d=ro
```

- A **primeira linha** define em quais teclados a regra vale (qualquer notebook Lenovo).
- A **segunda linha** diz que o scancode `9d` deve virar a tecla `ro`.

Salve com `Ctrl+O`, `Enter` e saia com `Ctrl+X`.

### 3. Aplicar a regra

```bash
sudo systemd-hwdb update
sudo udevadm trigger
```

- `systemd-hwdb update` recompila o banco de dados de hardware com a nova regra.
- `udevadm trigger` reaplica as regras aos dispositivos, sem precisar reiniciar.

### 4. Testar

Aperte a tecla `/ ?`. Para o `?`, use `Shift` + a tecla. Se não funcionar de imediato, reinicie o notebook.

A regra fica em `/etc/udev/hwdb.d/`, então continua valendo após reiniciar e após atualizações do sistema.

## Como desfazer

```bash
sudo rm /etc/udev/hwdb.d/90-teclado.hwdb
sudo systemd-hwdb update
sudo udevadm trigger
```

## Solução de problemas

| Problema | O que verificar |
|---|---|
| Aparece outro caractere em vez de `/` | O layout provavelmente não é Português (Brasil) |
| Nada muda após aplicar | Reinicie; confira se a segunda linha do arquivo começa com um espaço |
| O `showkey` não mostra nada ao apertar a tecla | Use o `evtest`; se também não mostrar, é problema físico |
| O scancode do seu teclado é diferente | Ajuste o valor em `KEYBOARD_KEY_xx`, lembrando de somar `0x80` quando houver prefixo `e0` |

## Resumo

| Etapa | Ação |
|---|---|
| Sintoma | Tecla `/ ?` não digita nada |
| Causa | A tecla envia o scancode `0xe0 0x1d` (Ctrl direito) |
| Correção | Regra no hwdb mapeando `9d` para `ro` |
| Comandos | `systemd-hwdb update` + `udevadm trigger` |

## Referências

- `man 7 hwdb`
- `man showkey`
- Arquivo do sistema `/usr/lib/udev/hwdb.d/60-keyboard.hwdb` (exemplos de remapeamento)
