# Screenshot Watcher

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
| `Print Screen` | `screenshot_watcher_wayland.py` | Tela cheia |
| `Shift + Print Screen` | `screenshot_watcher_area_wayland.py` | Seleção de área |

O script de área tem um detalhe que eu só adicionei depois de perceber o problema na prática: se a pessoa cancela a seleção (aperta Esc), o `gnome-screenshot` ainda "roda com sucesso" tecnicamente, só que sem gerar arquivo — e isso deixava pastas de dia vazias pra trás. Adicionei uma checagem que remove a pasta se ela ficou vazia depois da tentativa.

## Como instalar (o passo a passo que eu de fato segui)

Deixo aqui exatamente a ordem que funcionou pra mim, com os comandos que rodei. Testado em Ubuntu 22.04 LTS, sessão Wayland.

### 1. Verificar a sessão gráfica

Antes de qualquer coisa, confirma se você tá em Wayland ou Xorg, porque muda a estratégia:

```bash
echo $XDG_SESSION_TYPE
```

Se voltar `wayland` (padrão do Ubuntu 22.04 com GNOME), segue o guia normal abaixo. Se voltar `x11`, os scripts funcionam do mesmo jeito — a única diferença é que em Xorg você teria a opção adicional de usar `pynput` pra escutar o teclado direto, mas eu não vi motivo pra complicar quando o caminho do atalho nativo já resolve nos dois casos.

### 2. Instalar a dependência de sistema

O único requisito externo é o `gnome-screenshot`. No Ubuntu ele normalmente já vem instalado, mas confirma:

```bash
which gnome-screenshot || sudo apt install -y gnome-screenshot
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

## O que eu mudaria se fosse fazer de novo

Não gastaria tempo tentando fazer o `pynput` funcionar antes de checar `XDG_SESSION_TYPE`. Isso teria me poupado uns bons minutos de captura silenciosamente não funcionando e eu achando que era erro no meu código.

Também teria pensado desde o início em rodar via atalho do GNOME em vez de daemon. Não é só mais leve — é mais simples de manter, não tem processo pra reiniciar se travar, não precisa de `systemd`, não precisa de `loginctl enable-linger`. Às vezes a solução "chique" (serviço, systemd, reinício automático) é overengineering pra um problema que o próprio sistema operacional já resolve de graça.

Uma coisa que ainda não fiz, mas devia: testar em outra máquina com Xorg pra confirmar que o mesmo atalho personalizado funciona igual lá (acho que sim, mas "acho que sim" não é a mesma coisa que testar).
