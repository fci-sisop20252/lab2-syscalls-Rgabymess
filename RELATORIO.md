# 📝 Relatório do Laboratório 2 - Chamadas de Sistema

---

## 1️⃣ Exercício 1a - Observação printf() vs 1b - write()

### 💻 Comandos executados:
```bash
strace -e write ./ex1a_printf
strace -e write ./ex1b_write
```

### 🔍 Análise

**1. Quantas syscalls write() cada programa gerou?**
- ex1a_printf: __9___ syscalls
- ex1b_write: __7___ syscalls

**2. Por que há diferença entre os dois métodos? Consulte o docs/printf_vs_write.md**

```
No primeiro caso usamos printf(), que faz processamento interno (como formatação e buffering) antes de chamar write(). Isso pode resultar em múltiplas chamadas write() mesmo para uma única linha de código.
Já no segundo caso usamos diretamente write(), sem nenhuma intermediação, o que resulta em uma chamada de sistema por linha escrita no código.
```

**3. Qual método é mais previsível? Por quê você acha isso?**

```
O método mais previsível é o segundo, utilizando write(), porque cada chamada write() no código resulta exatamente em uma chamada de sistema, sem surpresas ou otimizações internas como acontece com printf().
```

---

## 2️⃣ Exercício 2 - Leitura de Arquivo

### 📊 Resultados da execução:
- File descriptor: __3___
- Bytes lidos: __127___

### 🔧 Comando strace:
```bash
strace -e openat,read,close ./ex2_leitura
```

### 🔍 Análise

**1. Qual file descriptor foi usado? Por que não começou em 0, 1 ou 2?**

```
O file descriptor usado foi o número 3 (ou outro número maior que 2). Isso acontece porque os descritores 0, 1 e 2 já são reservados pelo sistema para a entrada padrão (stdin), saída padrão (stdout) e saída de erro (stderr). Quando o programa chama open(), o sistema retorna o próximo número disponível, que é 3 ou superior.
```

**2. Como você sabe que o arquivo foi lido completamente?**

```
Sabemos que o arquivo foi lido completamente porque o número de bytes lidos foi menor ou igual ao tamanho do buffer (BUFFER_SIZE - 1), e o conteúdo do arquivo apareceu corretamente na saída. Além disso, read() não retornou erro (valor negativo) e o texto não foi cortado.
```

**3. Por que verificar retorno de cada syscall?**

```
Porque chamadas de sistema como open(), read() e close() podem falhar por vários motivos (arquivo inexistente, permissão negada, erro de hardware, etc.). Verificar o retorno evita que o programa continue com comportamentos inesperados e ajuda a identificar problemas mais facilmente.
```

---

## 3️⃣ Exercício 3 - Contador com Loop

### 📋 Resultados (BUFFER_SIZE = 64):
- Linhas: __25___ (esperado: 25)
- Caracteres: __1300___
- Chamadas read(): _21____
- Tempo: __0.000119___ segundos

### 🧪 Experimentos com buffer:

| Buffer Size | Chamadas read() | Tempo (s) |
|-------------|-----------------|-----------|
| 16          |       82        |  0.000491 |
| 64          |       21        |  0.000119 |
| 256         |        6        |  0.000120 |
| 1024        |        2        |  0.000216 |

### 🔍 Análise

**1. Como o tamanho do buffer afeta o número de syscalls?**

```
À medida que o tamanho do buffer aumenta, o número de chamadas read() necessárias para ler o arquivo completo diminui. Isso acontece porque buffers maiores conseguem ler mais dados por vez, reduzindo a quantidade de interações com o sistema operacional. Por exemplo, com buffer de 16 bytes foram necessárias 82 chamadas, enquanto com 1024 bytes apenas 2.
```

**2. Todas as chamadas read() retornaram BUFFER_SIZE bytes? Discorra brevemente sobre**

```
Não necessariamente. A função read() pode retornar menos bytes do que o tamanho do buffer por diversos motivos: o final do arquivo foi alcançado, o arquivo é menor que o buffer, ou por questões internas do sistema operacional. A última chamada, por exemplo, costuma retornar menos bytes porque há menos dados restantes no arquivo.
```

**3. Qual é a relação entre syscalls e performance?**

```
Cada syscall (como read()) envolve uma transição do modo usuário para o modo kernel, o que tem um custo de desempenho. Quanto mais syscalls, maior o tempo de execução total. Buffers menores exigem mais chamadas, aumentando esse custo. Já buffers maiores reduzem a quantidade de chamadas e, portanto, melhoram a performance — como mostrado pelos tempos na tabela.
```

---

## 4️⃣ Exercício 4 - Cópia de Arquivo

### 📈 Resultados:
- Bytes copiados: __1364___
- Operações: _6____
- Tempo: _0.000612____ segundos
- Throughput: __2176.52___ KB/s

### ✅ Verificação:
```bash
diff dados/origem.txt dados/destino.txt
```
Resultado: [V] Idênticos [F] Diferentes

### 🔍 Análise

**1. Por que devemos verificar que bytes_escritos == bytes_lidos?**

```
Verificamos isso para garantir que todos os dados lidos foram realmente escritos no arquivo de destino. A função write() pode escrever menos do que o número de bytes solicitado, especialmente em casos de erro, interrupções ou recursos limitados (como disco cheio).
Se não verificarmos essa igualdade, a cópia pode ficar incompleta ou corrompida, mesmo que o programa termine sem erro evidente.
```

**2. Que flags são essenciais no open() do destino?**

```
O_WRONLY | O_CREAT | O_TRUNC
```

**3. O número de reads e writes é igual? Por quê?**

```
Sim, no código implementado, o número de chamadas read() e write() é igual. Isso acontece porque, para cada bloco lido com read(), há uma chamada correspondente de write() com exatamente os mesmos dados.

Isso só é possível porque estamos tratando os dados em blocos fixos no laço de cópia, e não implementamos tratamento de escrita parcial.
```

**4. Como você saberia se o disco ficou cheio?**

```
Se o disco ficar cheio durante a cópia, a função write() não conseguirá escrever todos os dados e retornará -1. O programa então exibirá uma mensagem de erro através da função perror, como por exemplo: "Erro na escrita: No space left on device". Além disso, é possível observar esse erro diretamente no strace, onde a chamada write() falhará com o código de erro ENOSPC. Isso indicaria que não há mais espaço disponível para completar a escrita. Outro sinal seria o arquivo de destino ter tamanho menor do que o original, o que confirmaria que a cópia foi interrompida.
```

**5. O que acontece se esquecer de fechar os arquivos?**

```
Se os arquivos não forem fechados corretamente com close(), o sistema pode manter os descritores de arquivos abertos até o processo terminar. Isso pode causar vazamento de recursos, principalmente em programas maiores ou que manipulam muitos arquivos. Além disso, alguns dados podem não ser gravados corretamente no disco se ainda estiverem em buffers do sistema, o que pode comprometer a integridade do arquivo copiado. Em sistemas maiores ou de longa duração, esquecer de fechar arquivos pode levar ao esgotamento do número máximo de arquivos abertos por processo, gerando falhas difíceis de diagnosticar. Por isso, sempre se deve fechar os arquivos explicitamente ao final do uso.
```

---

## 🎯 Análise Geral

### 📖 Conceitos Fundamentais

**1. Como as syscalls demonstram a transição usuário → kernel?**

```
As system calls (syscalls) representam o ponto de contato entre programas em modo usuário e o sistema operacional, que roda em modo kernel. Sempre que um programa realiza uma operação que exige acesso a recursos do sistema (como arquivos, rede ou memória), ele precisa fazer uma syscall. Durante esse processo, o controle é transferido do modo usuário para o modo kernel, onde a operação é executada com permissões elevadas. O strace é uma ferramenta que nos permite observar essas transições em tempo real, mostrando exatamente quando e como o programa acessa o kernel por meio de chamadas como open(), read(), write() e close().
```

**2. Qual é o seu entendimento sobre a importância dos file descriptors?**

```
File descriptors (FDs) são inteiros usados pelo sistema operacional para identificar arquivos abertos (e outros recursos como sockets e pipes) dentro de um processo. Eles são essenciais porque abstraem o acesso a qualquer recurso de E/S, permitindo que o programa interaja com arquivos de forma unificada, independentemente do tipo de dispositivo. Por padrão, os FDs 0, 1 e 2 representam a entrada padrão (stdin), saída padrão (stdout) e erro padrão (stderr). Sem o uso de file descriptors, seria muito mais difícil gerenciar múltiplos arquivos ou fluxos de dados de forma eficiente e organizada dentro de um programa.
```

**3. Discorra sobre a relação entre o tamanho do buffer e performance:**

```
O tamanho do buffer tem impacto direto na performance de operações de entrada/saída. Buffers pequenos geram mais chamadas read() e write(), aumentando a quantidade de transições entre o modo usuário e o modo kernel — o que adiciona overhead ao programa. Já buffers maiores reduzem o número de syscalls, permitindo que mais dados sejam transferidos por chamada, o que melhora o throughput (bytes/s). No entanto, buffers muito grandes podem desperdiçar memória ou não trazer ganhos significativos se o sistema já estiver limitado por outros fatores. O ideal é encontrar um tamanho de buffer que equilibre bem consumo de memória e número de chamadas de sistema.
```

### ⚡ Comparação de Performance

```bash
# Teste seu programa vs cp do sistema
time ./ex4_copia
time cp dados/origem.txt dados/destino_cp.txt
```

**Qual foi mais rápido?** __cp do sistema___

**Por que você acha que foi mais rápido?**

```
O comando cp é implementado em baixo nível e altamente otimizado, utilizando buffers maiores e técnicas avançadas como leitura assíncrona, mmap, ou otimizações específicas para o sistema de arquivos. Já o programa ex4_copia, embora funcional, é mais simples e usa buffers menores, além de fazer uma syscall read() e write() por vez, o que gera mais transições de modo usuário para modo kernel. Essa diferença de implementação explica por que o cp costuma ser mais rápido, mesmo realizando a mesma tarefa.
```

---

## 📤 Entrega
Certifique-se de ter:
- [ ] Todos os códigos com TODOs completados
- [ ] Traces salvos em `traces/`
- [ ] Este relatório preenchido como `RELATORIO.md`

```bash
strace -e write -o traces/ex1a_trace.txt ./ex1a_printf
strace -e write -o traces/ex1b_trace.txt ./ex1b_write
strace -o traces/ex2_trace.txt ./ex2_leitura
strace -c -o traces/ex3_stats.txt ./ex3_contador
strace -o traces/ex4_trace.txt ./ex4_copia
```
# Bom trabalho!
