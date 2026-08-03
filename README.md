# # Screenshot Watcher

Um script Python pequeno que resolve um problema que eu tinha todo dia: prints jogados soltos na pasta `Imagens`, sem organização nenhuma, misturados uns com os outros até eu não saber mais o que era o quê nem de quando.

A ideia é simples. Toda vez que aperto Print Screen, o print cai automaticamente numa pasta com a data de hoje, com o horário no nome do arquivo. Nada de "Captura de tela de 2026-05-23 (47).png" duplicado ou sobrescrito.

## Por que eu fiz isso

Uso muito print pra documentar bug, mandar pro time, guardar referência visual de alguma coisa. Depois de meses, minha pasta de capturas virou uma bagunça com centenas de arquivos soltos, sem nenhuma estrutura. Cansei de procurar "aquele print que eu tirei semana passada" rolando uma lista infinita.

Dava pra resolver isso com qualquer script de 10 linhas. O que eu queria, na real, era aprender a fazer isso do jeito certo: rodando em segundo plano (ou nem isso, como vocês vão ver), sem comer recurso do sistema, e entendendo de verdade como o Linux lida com captura de teclado e de tela.

## A primeira ideia (que não funcionou)

Meu plano inicial era o clássico: usar `pynput` pra escutar o teclado globalmente e disparar a captura com `scrot` quando detectasse o Print Screen. Solução direta, biblioteca conhecida, achei que ia ser tranquilo.

Não foi. Testei e a tecla simplesmente não era capturada. Rodei um `echo $XDG_SESSION_TYPE` e vi: `wayland`. O Ubuntu 22.04 já vem com Wayland como sessão padrão do GNOME, e Wayland bloqueia por design que aplicativos escutem teclas do sistema inteiro — é uma proteção contra keyloggers, faz sentido de segurança, mas quebra completamente esse tipo de abordagem.

Cheguei a cogitar trocar a sessão de login pra Xorg só pra fazer o `pynput` funcionar. Ia funcionar, mas ia ser bater fora do prumo do sistema todo por causa de um script de screenshot. Não valia o custo.

## O pivô: deixar o GNOME fazer o trabalho pesado

Virei a lógica de cabeça pra baixo. Em vez de eu tentar escutar o teclado por fora, usei o que o GNOME já faz nativamente: atalhos de teclado personalizados, configurados direto em **Configurações → Teclado → Atalhos personalizados**. O sistema já processa a tecla de qualquer jeito; só precisei apontar esse atalho pro meu script.

Isso resolveu dois problemas de uma vez. Primeiro, o bloqueio do Wayland deixou de existir, porque quem intercepta a tecla é o próprio GNOME Shell, não um processo meu. Segundo — e isso eu só percebi depois, meio sem querer — o script deixou de precisar ficar rodando em background o tempo todo. Ele só executa no exato momento em que a tecla é pressionada. Fiquei com um `systemd` service pronto (tem no histórico de commits, se alguém quiser ver) e simplesmente não usei. Zero processo residente, zero RAM ocupada em repouso. Isso é mais leve do que qualquer daemon bem otimizado que eu poderia ter escrito.

Pra captura em si, troquei `scrot` por `gnome-screenshot`. `scrot` depende de X11/XShm e não funciona em Wayland. `gnome-screenshot` já fala com o portal certo e funciona nos dois ambientes.

## Um tropeço bobo no meio do caminho

Depois de escrever o script, tentei rodar ele com `python3 ~/scripts/.../arquivo.py` e recebi `No such file or directory`. Óbvio, em retrospecto: eu tinha criado o arquivo em outro lugar (numa conversa com IA, gerando o código) e nunca tinha, de fato, colocado ele no caminho certo da minha máquina. Ninguém tinha copiado nada pra lá.

Resolvi colando o conteúdo direto com `cat > arquivo.py << 'EOF' ... EOF` no terminal. Meio old school, mas funcionou de primeira e eu pelo menos tenho certeza de que o arquivo existe onde eu preciso que ele exista.

## Estrutura final

```
~/Imagens/Capturas de tela/
├── 29-07-2026/
│   ├── print_14-32-05.png
│   └── print_18-01-22.png
└── 30-07-2026/
    ├── print_09-15-40.png
    └── print_area_09-16-02.png
```

Duas capturas, dois atalhos:

| Atalho | Script | O que faz |
|---|---|---|
| `Print Screen` | `screenshot_watcher_wayland.py` | Tela cheia, salva e copia pro clipboard |
| `Shift + Print Screen` | `screenshot_watcher_area_wayland.py` | Seleção de área, salva e copia pro clipboard |

O script de área tem um detalhe que eu só adicionei depois de perceber o problema na prática: se a pessoa cancela a seleção (aperta Esc), o `gnome-screenshot` ainda "roda com sucesso" tecnicamente, só que sem gerar arquivo — e isso deixava pastas de dia vazias pra trás. Adicionei uma checagem que remove a pasta se ela ficou vazia depois da tentativa.

(A coluna "copia pro clipboard" é uma atualização posterior — conto a história dela mais abaixo, na seção **Update: o Ctrl+V que sumiu**.)

## Como instalar (o passo a passo que eu de fato segui)

Deixo aqui exatamente a ordem que funcionou pra mim, com os comandos que rodei. Testado em Ubuntu 22.04 LTS, sessão Wayland.

### 1. Verificar a sessão gráfica

Antes de qualquer coisa, confirma se você tá em Wayland ou Xorg, porque muda a estratégia:

```bash
echo $XDG_SESSION_TYPE
```

Se voltar `wayland` (padrão do Ubuntu 22.04 com GNOME), segue o guia normal abaixo. Se voltar `x11`, os scripts funcionam do mesmo jeito — a única diferença é que em Xorg você teria a opção adicional de usar `pynput` pra escutar o teclado direto, mas eu não vi motivo pra complicar quando o caminho do atalho nativo já resolve nos dois casos.

### 2. Instalar as dependências de sistema

São duas: `gnome-screenshot`, pra capturar, e `wl-clipboard`, pra copiar o resultado pra área de transferência (essa segunda entrou depois, na atualização que descrevo lá embaixo — mas já deixo aqui pra quem for instalar do zero não precisar voltar duas vezes).

```bash
which gnome-screenshot || sudo apt install -y gnome-screenshot
sudo apt install -y wl-clipboard
```

Não precisei instalar nenhum pacote Python. Cheguei a cogitar `Pillow` no começo (pra usar `ImageGrab`), mas descartei — ia adicionar uma dependência externa pra fazer exatamente o que o `gnome-screenshot` já faz de graça e nativamente integrado ao Wayland.

### 3. Criar a pasta do projeto

```bash
mkdir -p ~/scripts/screenshot_watcher
cd ~/scripts/screenshot_watcher
```

### 4. Criar os dois scripts

Aqui foi onde eu tropecei da primeira vez: gerei o código, tentei rodar `python3 ~/scripts/.../arquivo.py` e levei um `No such file or directory`, porque o arquivo nunca tinha sido efetivamente salvo naquele caminho — só existia como texto solto. A saída que funcionou foi colar o conteúdo direto no terminal com `cat`:

```bash
cat > ~/scripts/screenshot_watcher/screenshot_watcher_wayland.py << 'EOF'
# (conteúdo do script de tela cheia — arquivo completo no repositório)
EOF

cat > ~/scripts/screenshot_watcher/screenshot_watcher_area_wayland.py << 'EOF'
# (conteúdo do script de área — arquivo completo no repositório)
EOF
```

Os arquivos completos estão em [`screenshot_watcher_wayland.py`](./screenshot_watcher_wayland.py) e [`screenshot_watcher_area_wayland.py`](./screenshot_watcher_area_wayland.py) neste repositório. Copia o conteúdo de lá se for repetir o processo.

Depois, permissão de execução (não é estritamente obrigatório já que sempre chamei via `python3 arquivo.py`, mas deixei configurado por hábito):

```bash
chmod +x ~/scripts/screenshot_watcher/*.py
```

### 5. Testar cada script isoladamente, sem atalho nenhum

Isso foi importante pra mim: só configurei o atalho de teclado depois de confirmar que o script funcionava sozinho, rodando manualmente. Economiza tempo de debug — se não funcionar no terminal, não vai funcionar via atalho também, e é mais fácil ver a mensagem de erro direto no console.

```bash
python3 ~/scripts/screenshot_watcher/screenshot_watcher_wayland.py
```

Conferi o resultado com:

```bash
ls -la ~/Imagens/"Capturas de tela"/$(date +%d-%m-%Y)/
```

Pra testar o script de área:

```bash
python3 ~/scripts/screenshot_watcher/screenshot_watcher_area_wayland.py
```

O cursor vira uma mira de seleção, arrasta pra escolher a região, solta — e confere na mesma pasta.

### 6. Remover os atalhos padrão do GNOME

Esse passo é fácil de esquecer e, se pular, os dois sistemas (o seu script e o utilitário nativo do GNOME) disparam ao mesmo tempo na mesma tecla.

1. **Configurações → Teclado → Ver e personalizar atalhos → Capturas de tela.**
2. Vão aparecer entradas como "Salvar uma captura de tela em Imagens" e "Mostrar a interface de captura de tela", já associadas ao `Print Screen` e possivelmente ao `Shift+Print Screen`.
3. Clica em cada uma e aperta **Backspace** pra limpar o atalho.

### 7. Criar os atalhos personalizados

Ainda em **Teclado**, desce até **Atalhos personalizados** e clica em **+**. Fiz um pra cada script:

**Atalho 1 — tela cheia**
- Nome: `Screenshot Watcher`
- Comando: `python3 /home/charlie/scripts/screenshot_watcher/screenshot_watcher_wayland.py`
- Tecla: `Print Screen`

**Atalho 2 — área selecionada**
- Nome: `Screenshot Watcher - Área`
- Comando: `python3 /home/charlie/scripts/screenshot_watcher/screenshot_watcher_area_wayland.py`
- Tecla: `Shift + Print Screen`

Troca `/home/charlie/` pelo caminho real do seu usuário.

### 8. Teste final, sem terminal nenhum aberto

Aperta `Print Screen` puro, depois `Shift+Print Screen`, e confere a pasta do dia:

```bash
ls -la ~/Imagens/"Capturas de tela"/$(date +%d-%m-%Y)/
```

Se os dois arquivos apareceram (`print_HH-MM-SS.png` e `print_area_HH-MM-SS.png`), tá funcionando exatamente como deveria — e sem nenhum processo Python residente rodando em segundo plano esperando por isso.

## Update: o Ctrl+V que sumiu

Usei o script assim por uns dias e só fui perceber o que tinha perdido quando fui colar um print direto num chat de IA, do jeito que sempre fiz, e não colou nada. Tive que parar, abrir o gerenciador de arquivos, procurar a pasta do dia, achar o print certo, arrastar. Justo o tipo de fricção que o projeto inteiro nasceu pra eliminar — só que numa etapa diferente do fluxo.

O atalho padrão do GNOME, aquele que eu substituí, copiava a imagem pro clipboard junto com salvar (ou só copiava, dependendo de qual das duas opções nativas eu tinha usado antes). O `gnome-screenshot -f` que eu chamo no script só salva o arquivo. Ele nunca tocou no clipboard, e eu simplesmente não tinha percebido, porque nos primeiros testes eu só chequei se o arquivo tinha sido criado — não testei o fluxo de colar em outro lugar.

A correção não exigiu trocar de ferramenta, só somar uma etapa depois do `gnome-screenshot` salvar: jogar o conteúdo do PNG recém-criado pro clipboard usando `wl-copy` (do pacote `wl-clipboard`).

```python
def copiar_para_clipboard(caminho_arquivo: Path) -> None:
    with open(caminho_arquivo, "rb") as f:
        subprocess.run(
            ["wl-copy", "--type", "image/png"],
            stdin=f,
            check=True,
        )
```

Chamei essa função logo depois de confirmar que o arquivo existe, tanto no script de tela cheia quanto no de área. Simples assim — ler o arquivo que acabei de salvar e mandar pro `wl-copy`.

Dois detalhes que valem registrar, porque eu mesmo não sabia até esbarrar neles:

**O `--type image/png` não é opcional.** Sem isso, dependendo do app onde eu colava, a imagem virava um caminho de arquivo em texto plano em vez da imagem em si. O `wl-copy` precisa saber o mime type pra anunciar corretamente pros outros programas o que tá disponível pra colar.

**O `wl-copy` não é um comando que "roda e acaba".** No Wayland, diferente do X11, não existe um clipboard manager central guardando tudo que foi copiado — quem segura o conteúdo é o próprio processo que copiou, e ele precisa continuar vivo pra responder quando alguém colar. Então o `wl-copy` se destaca do processo pai e continua rodando sozinho em segundo plano depois que meu script termina. No começo isso me deixou com um pé atrás — "ué, mas eu não queria processo nenhum residente" — mas é comportamento esperado da ferramenta, não um vazamento nem nada que eu escrevi errado. Ele fica inerte até alguém colar ou até um novo conteúdo substituir o que tá no clipboard, que aí ele mesmo encerra.

Não escrevi tratamento de erro chique pra esse passo. Se o `wl-copy` não estiver instalado, o script avisa no stderr e segue em frente — o arquivo já foi salvo de qualquer forma, então a pior consequência de falhar aqui é eu ter que abrir a pasta manualmente de novo, exatamente como antes da atualização. Não é motivo pra derrubar o script inteiro com `sys.exit()`.

### Os comandos que rodei pra aplicar a atualização

Segui o mesmo método do primeiro dia: sobrescrevi os dois arquivos direto no terminal com `cat`, em vez de abrir um editor. Já tinha o hábito, e assim eu garanto que o conteúdo que está no meu disco é exatamente o que eu pretendia, sem risco de colar errado num editor gráfico.

Primeiro, o script de tela cheia:

```bash
cat > ~/scripts/screenshot_watcher/screenshot_watcher_wayland.py << 'EOF'
#!/usr/bin/env python3
"""
Screenshot Watcher (versão Wayland) - tela cheia
--------------------------------------------------
Executado sob demanda pelo GNOME Shell via atalho de teclado (Print Screen).

Além de salvar em:
    ~/Imagens/Capturas de tela/DD-MM-AAAA/print_HH-MM-SS.png

também copia a imagem para a área de transferência, permitindo colar
diretamente (Ctrl+V) em qualquer aplicativo, sem precisar abrir a pasta.
"""

import subprocess
import sys
from pathlib import Path
from datetime import datetime

BASE_DIR = Path.home() / "Imagens" / "Capturas de tela"


def copiar_para_clipboard(caminho_arquivo: Path) -> None:
    """Copia o conteúdo do PNG para a área de transferência via wl-copy."""
    try:
        with open(caminho_arquivo, "rb") as f:
            subprocess.run(
                ["wl-copy", "--type", "image/png"],
                stdin=f,
                check=True,
            )
    except FileNotFoundError:
        print("Aviso: 'wl-copy' não instalado. "
              "Rode: sudo apt install wl-clipboard", file=sys.stderr)
    except subprocess.CalledProcessError as e:
        print(f"Aviso: falha ao copiar para a área de transferência: {e}",
              file=sys.stderr)


def capturar_tela() -> None:
    agora = datetime.now()

    pasta_dia = BASE_DIR / agora.strftime("%d-%m-%Y")
    pasta_dia.mkdir(parents=True, exist_ok=True)

    nome_arquivo = agora.strftime("print_%H-%M-%S.png")
    caminho_completo = pasta_dia / nome_arquivo

    try:
        subprocess.run(
            ["gnome-screenshot", "-f", str(caminho_completo)],
            check=True,
            stdout=subprocess.DEVNULL,
            stderr=subprocess.DEVNULL,
        )
    except FileNotFoundError:
        print("Erro: 'gnome-screenshot' não instalado. "
              "Rode: sudo apt install gnome-screenshot", file=sys.stderr)
        sys.exit(1)
    except subprocess.CalledProcessError as e:
        print(f"Erro ao capturar tela: {e}", file=sys.stderr)
        sys.exit(1)

    if caminho_completo.exists():
        copiar_para_clipboard(caminho_completo)


if __name__ == "__main__":
    capturar_tela()
EOF
```

Segundo, o script de área selecionada:

```bash
cat > ~/scripts/screenshot_watcher/screenshot_watcher_area_wayland.py << 'EOF'
#!/usr/bin/env python3
"""
Screenshot Watcher - Captura de área selecionada (versão Wayland)
--------------------------------------------------------------------
Disparado sob demanda pelo GNOME Shell (atalho Shift+Print Screen).
Abre a ferramenta de seleção de área do GNOME, salva o resultado em:

    ~/Imagens/Capturas de tela/DD-MM-AAAA/print_area_HH-MM-SS.png

e copia a imagem para a área de transferência, permitindo colar (Ctrl+V)
direto em qualquer aplicativo, sem precisar abrir a pasta.

Se o usuário cancelar a seleção (Esc), nenhum arquivo é criado e nada
é copiado.
"""

import subprocess
import sys
from pathlib import Path
from datetime import datetime

BASE_DIR = Path.home() / "Imagens" / "Capturas de tela"


def copiar_para_clipboard(caminho_arquivo: Path) -> None:
    """Copia o conteúdo do PNG para a área de transferência via wl-copy."""
    try:
        with open(caminho_arquivo, "rb") as f:
            subprocess.run(
                ["wl-copy", "--type", "image/png"],
                stdin=f,
                check=True,
            )
    except FileNotFoundError:
        print("Aviso: 'wl-copy' não instalado. "
              "Rode: sudo apt install wl-clipboard", file=sys.stderr)
    except subprocess.CalledProcessError as e:
        print(f"Aviso: falha ao copiar para a área de transferência: {e}",
              file=sys.stderr)


def capturar_area() -> None:
    agora = datetime.now()

    pasta_dia = BASE_DIR / agora.strftime("%d-%m-%Y")
    pasta_dia.mkdir(parents=True, exist_ok=True)

    nome_arquivo = agora.strftime("print_area_%H-%M-%S.png")
    caminho_completo = pasta_dia / nome_arquivo

    try:
        subprocess.run(
            ["gnome-screenshot", "-a", "-f", str(caminho_completo)],
            check=True,
            stdout=subprocess.DEVNULL,
            stderr=subprocess.DEVNULL,
        )
    except FileNotFoundError:
        print("Erro: 'gnome-screenshot' não instalado. "
              "Rode: sudo apt install gnome-screenshot", file=sys.stderr)
        sys.exit(1)
    except subprocess.CalledProcessError as e:
        print(f"Erro ao capturar área: {e}", file=sys.stderr)
        sys.exit(1)

    if caminho_completo.exists():
        copiar_para_clipboard(caminho_completo)
    else:
        try:
            pasta_dia.rmdir()
        except OSError:
            pass


if __name__ == "__main__":
    capturar_area()
EOF
```
-----------------------------------------------------------------
Testes:
```bash
python3 ~/scripts/screenshot_watcher/screenshot_watcher_wayland.py
```
Abre qualquer app (editor de texto, navegador, o que for) e dá Ctrl+V. A imagem deve colar direto, sem precisar abrir a pasta.

Repete o teste com o de área:

```bash
python3 ~/scripts/screenshot_watcher/screenshot_watcher_area_wayland.py
```

Seleciona uma região, solta, e testa o Ctrl+V de novo.


----------------------------------------------------------------

Reparei numa coisa depois de rodar os dois: como uso `cat > ` (com um `>` só, não `>>`), o conteúdo antigo do arquivo é completamente substituído. Isso é exatamente o que eu queria aqui, mas é o tipo de detalhe que, se eu não tivesse prestado atenção, podia ter me feito perder alguma customização que eu tivesse feito manualmente nos scripts entre uma sessão e outra. Não foi o caso, mas anotei mentalmente pra próxima vez conferir um `diff` antes de sobrescrever, se o arquivo já tiver histórico de edições manuais.

Depois de rodar os dois comandos, não precisei tocar em nada nos atalhos do GNOME — o comando configurado neles (`python3 /caminho/do/script.py`) continua o mesmo, só o conteúdo por trás mudou. Testei igual descrevi acima: rodando cada script direto no terminal primeiro, e só depois confiando no atalho de teclado.

## O que eu mudaria se fosse fazer de novo

Não gastaria tempo tentando fazer o `pynput` funcionar antes de checar `XDG_SESSION_TYPE`. Isso teria me poupado uns bons minutos de captura silenciosamente não funcionando e eu achando que era erro no meu código.

Também teria pensado desde o início em rodar via atalho do GNOME em vez de daemon. Não é só mais leve — é mais simples de manter, não tem processo pra reiniciar se travar, não precisa de `systemd`, não precisa de `loginctl enable-linger`. Às vezes a solução "chique" (serviço, systemd, reinício automático) é overengineering pra um problema que o próprio sistema operacional já resolve de graça.

Uma coisa que ainda não fiz, mas devia: testar em outra máquina com Xorg pra confirmar que o mesmo atalho personalizado funciona igual lá (acho que sim, mas "acho que sim" não é a mesma coisa que testar).

E a lição mais chata de admitir: da próxima vez eu testo o fluxo de uso completo antes de considerar a feature pronta, não só o resultado técnico isolado. "O arquivo foi criado" e "o arquivo foi criado do jeito que eu realmente uso no dia a dia" são coisas diferentes, e eu só descobri isso alguns dias depois, no meio de um chat, tentando colar um print que não vinha.
