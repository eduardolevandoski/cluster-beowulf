# Cluster Beowulf

### 1° Passo - Criação das máquinas virtuais

  Para realizar a configuração, iremos utilizar a imagem mais recente ubuntu
server 24. Após importar as 3 máquinas virtuais (master, node1 e node2), é
necessário configurar os adaptadores de rede delas no Virtual Box. Em cada
máquina, utilizaremos o adaptador 1 como “NAT” para permitir comunicação com a
internet e o adaptador 2 como “Rede Interna” para que as VMs possam se
comunicar.

<p align="center">
  <img src="https://github.com/user-attachments/assets/d57b977e-e9bf-4bc9-af7e-55e2cbafed96" alt="Centered Image">
</p>

### 2° Passo - Configuração da rede

  Depois disso, é necessário configurar a rede de cada máquina abrindo o
arquivo 50-cloud-init.yaml (sudo nano /etc/netplan/50-cloud-init.yaml), e deixando
o adaptador 1 (enp0s3) com o dhcp ativado e o adaptador 2 (enp0s8) com um ip fixo
(os ips de todas as máquinas devem pertencer à mesma rede). Eis um exemplo de
conteúdo do arquivo:

<p align="center">
  <img src="https://github.com/user-attachments/assets/d1332905-2cd0-4002-8872-971baca351cb" alt="Centered Image">
</p>

### 3° Passo - Instalação dos softwares necessários

  Feita a configuração inicial da rede, precisamos instalar as bibliotecas
necessárias e configurar o acesso SSH para rodar o cluster. Para isso podemos
utilizar os seguintes comandos em cada máquina:

```bash
sudo apt update
sudo apt install openmpi-bin openmpi-common libopenmpi-dev ssh
```

  Com isso, teremos instaladas as ferramentas necessárias para que o cluster
funcione corretamente.

### 4° Passo - Configuração do acesso SSH

  O próximo passo é configurar o acesso SSH entre as máquinas, para que a
máquina master possa acessar os nós para gerenciar o cluster. Para isso, devemos
gerar uma chave SSH na máquina master com o comando:

```bash
ssh-keygen -t rsa
```

  Após isso, precisamos copiar a chave pública para cada um dos nós de
processamento. Isso é pode ser feito com o seguinte comando comando:

```bash
ssh-copy-id <user_node>@<ip_node>
```

### 5° Passo - Execução de programas no cluster

  Para que possamos executar programas no nosso cluster de maneira
distribuída, utilizaremos MPI (Message Passing Interface) que é um padrão para
comunicação de dados em computação paralela. Para facilitar a execução,
podemos criar um arquivo (nano ~/hosts) na máquina master listando os nós de
processamento (incluindo o próprio master). O conteúdo do arquivo segue o
seguinte padrão:

```bash
<user_node>@<ip_node> slots=1
<user_node>@<ip_node> slots=1
<user_node>@<ip_node> slots=1
```

  Além disso, devemos colocar em cada um dos nós o programa que será
executado paralelamente. Esse programa executável deve estar no mesmo diretório
em todas as máquinas (ex.: /cluster/programa). Uma boa prática também seria
utilizar um sistema de arquivos NFS para compartilhamento do programa entre
todos os nós sem precisar copiá-lo para cada uma das máquinas. O programa que
utilizaremos para testar o cluster é um programa que calcula se um número é primo
ou não:

```c
#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>
#include <math.h>

// Função para verificar se n é divisível por qualquer número entre start e end
int is_divisible_in_range(int n, int start, int end) {
    int found_divisor = 0;
    for (int i = start; i <= end; i++) {
        if (n % i == 0) {
            found_divisor = 1;
            break;
        }
    }
    return found_divisor;
}

int main(int argc, char** argv) {
    int rank, size, n, intervalo, start, end, has_divisor, global_result;

    // Inicialização do MPI
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    if (rank == 0) {
        // O nó master (rank 0) pede o número via input
        printf("Digite um número para verificar se é primo: ");
        scanf("%d", &n);
    }

    // Envia o número para todos os processos
    MPI_Bcast(&n, 1, MPI_INT, 0, MPI_COMM_WORLD);

    // Divide o trabalho
    intervalo = n / size;

    // Define as variáveis de início e fim para cada processo
    start = rank * intervalo + 2;
    end = (rank + 1) * intervalo + 1;
    if (rank == size - 1) {
        end = n;
    }

    // Cada processo verifica se há divisores em seu intervalo
    has_divisor = is_divisible_in_range(n, start, end);

    // Coleta os resultados de todos os nós
    MPI_Reduce(&has_divisor, &global_result, 1, MPI_INT, MPI_LOR, 0, MPI_COMM_WORLD);

    // O nó master decide se o número é primo
    if (rank == 0) {
        if (global_result == 1) {
            printf("%d não é primo.\n", n);
        } else {
            printf("%d é primo.\n", n);
        }
    }

    MPI_Finalize();
    return 0;
}
```

  Para compilar o programa, utilizamos o seguinte comando:

```bash
mpicc -o /caminho/output /caminho/arquivo.c -lm
```

  Tendo o programa compilado e presente em todas as máquinas, podemos executá-lo em nosso cluster. 
Para isso, utilizaremos o seguinte comando:

```bash
mpirun --hostfile /caminho/arquivo/hosts -np <num_nodes> /caminho/do/programa
```

#

### Arquivo PDF e a apresentação
- [PDF](https://drive.google.com/file/d/1kinT3cVLRgUHiRyhn8MA8xjh0qBCX0fP/view?usp=drive_link)
- [Apresentação](https://drive.google.com/file/d/1ORBvL_Tgrl21wPzEzxAYy-J-TApala8_/view?usp=drive_link)

