# Configuração do Ambiente de Desenvolvimento no Linux

Este guia detalha os passos para instalar todas as ferramentas necessárias para compilar e testar o UFSKernel em um sistema operacional Linux.

## 1. Instalação do Toolchain ARM e QEMU

O primeiro passo é instalar o compilador cruzado (`cross-compiler`) para ARM, o emulador QEMU e o depurador GDB (opcional).

O compilador cruzado será utilizado para gerar binários para o nosso processador ARM na nossa máquina local x86. O QEMU será resposável por emular uma máquina virtual ARM para que possamos desenvolver e testar de forma simples e rápida. Já o GDB é uma ferramenta que permite a análise das instruções individuais que o processador virtual roda, facilitando a depuração em caso de erro.

Abra seu terminal e execute o seguinte comando:

```bash
sudo apt update
sudo apt install gcc-arm-none-eabi qemu-system-arm gdb-multiarch
```

Em Arch, `gcc-arm-none-eabi` está no AUR, enquanto `gdb` e `qemu-system-arm` podem ser instalados com `pacman` mesmo.

Instruções para demais distribuições são bem vindas!

### Verificando a Instalação

Após a instalação, verifique se cada componente foi instalado corretamente executando os seguintes comandos. Cada um deve imprimir a versão do programa sem erros de "command not found".

```bash
arm-none-eabi-gcc --version
qemu-system-arm --version
gdb-multiarch --version
```

## 2. Compilando e Executando o Kernel pela Primeira Vez

Com as ferramentas instaladas, você pode clonar o repositório do kernel e fazer o primeiro teste.

1.  **Clone o repositório do kernel:**
    ```bash
    git clone https://github.com/UFSKernel/kernel.git
    cd kernel
    ```

2.  **Compile e execute:**
    O `Makefile` do projeto automatiza todo o processo. Para compilar e rodar o kernel no QEMU, basta executar:
    ```bash
    make run
    ```

Se tudo estiver configurado corretamente, você deverá ver a seguinte saída no seu terminal:

```
Bem-vindo ao UFSKernel!
Executando em modo ARM bare-metal no QEMU.
```

O QEMU ficará em execução (em um loop infinito). Para pará-lo, é necessário matar o processo do QEMU (`pkill qemu-system-arm` ou `kill -9 <id_do_processo>`), pois não temos um tratador de CTRL + C.

## 3. (Opcional) Configurando o Debug com GDB

Para uma depuração mais avançada, você pode executar o kernel em modo de debug e conectar o GDB a ele.

1.  **Em um terminal, inicie o QEMU em modo de debug:**
    O QEMU iniciará, mas ficará "congelado", esperando pela conexão do GDB.
    ```bash
    make debug
    ```

2.  **Em um segundo terminal, inicie o GDB e conecte-se ao QEMU:**
    Navegue até a mesma pasta do kernel e execute:
    ```bash
    gdb-multiarch -ex "target remote localhost:1234" -ex "symbol-file kernel.elf"
    ```
3.  **Comece a depurar!**
    Dentro do GDB, você tem um ambiente poderoso para inspecionar e controlar a execução do seu kernel.
    
    #### Depuração de Código C (Nível de Linha)
    
    Estes são os comandos mais comuns para depurar seu código C:
    
    -   `continue` (ou `c`): Inicia ou continua a execução do kernel até o próximo ponto de parada (breakpoint).
    -   `break kmain` (ou `b kmain`): Cria um ponto de parada no início da função `kmain`. Você pode usar `b <nome_da_funcao>` ou `b <arquivo.c>:<numero_da_linha>`.
    -   `step` (ou `s`): Executa a próxima **linha** de código C. Se a linha for uma chamada de função, ele entra *dentro* da função.
    -   `next` (ou `n`): Similar ao `step`, mas se a linha for uma chamada de função, ele a executa por completo e para na linha *seguinte*, sem entrar nela.
    -   `print var_name` (ou `p var_name`): Imprime o valor atual de uma variável.
    -   `list` (ou `l`): Mostra as linhas de código fonte ao redor do ponto de parada atual.
    
    #### Depuração em Nível de Assembly (Nível de Instrução)
    
    Para entender o que o processador está *realmente* fazendo, você pode depurar instrução por instrução:
    
    -   **`layout asm`**: Ativa a visualização TUI (Text User Interface) do GDB, mostrando o código assembly em uma janela separada. Pressione `Ctrl + X` e depois `A` para alternar entre as visualizações.
    -   **`layout regs`**: Adiciona uma janela mostrando o estado de todos os registradores (como `r0`, `sp`, `pc`, `cpsr`), que é atualizada a cada passo. Essencial para depuração de baixo nível.
    -   **`stepi` (ou `si`)**: Executa uma **única instrução** de assembly. Se a instrução for um `bl` (chamada de função), ele entra nela.
    -   **`nexti` (ou `ni`)**: Executa uma **única instrução** de assembly, mas trata chamadas de função como uma única instrução (não entra nelas).
    -   **`info registers` (ou `i r`)**: Imprime o valor de todos os registradores. Você pode especificar registradores: `i r r0 sp pc`.
    -   **`disassemble` (ou `disas`)**: Mostra o código assembly da função atual.
    -   **`x/<N>xw <endereço>`**: O comando `examine` (`x`) é usado para inspecionar a memória. Por exemplo, `x/16xw 0x40000000` mostra 16 "words" (palavras de 4 bytes) em formato hexadecimal (`x`) a partir do endereço `0x40000000`.
    
    #### Dica de Fluxo de Trabalho: Inspecionando a Inicialização
    
    Para ver a mágica do `startup.S` acontecendo:
    1.  Inicie o QEMU com `make debug`. Ele ficará congelado.
    2.  Inicie o GDB.
    3.  No prompt do GDB, digite `layout regs` para ver os registradores.
    4.  Use o comando `si` para executar a primeira instrução (`mrs r0, cpsr`).
    5.  Use `si` novamente. Observe no painel de registradores como o valor de `r0` muda a cada passo (`bic`, `orr`).
    6.  Após a instrução `msr cpsr, r0`, você verá os bits de modo no registrador `cpsr` mudarem!!
    7.  Após `ldr sp, =stack_top`, você verá o registrador `sp` ser preenchido com o endereço do topo da pilha.
