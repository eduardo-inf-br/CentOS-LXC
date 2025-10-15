# Implantação de Contêiner LXC no CentOS 7.

## Introdução

Este guia detalhado fornecerá instruções passo a passo para a implantação e gerenciamento de contêineres Linux (LXC) em um sistema CentOS 7. LXC oferece uma solução de virtualização leve e eficiente, permitindo que múltiplos sistemas Linux isolados operem em um único host. Diferente das máquinas virtuais tradicionais, o LXC utiliza o kernel do sistema host, o que resulta em menor sobrecarga e maior desempenho. Ao final deste guia, você será capaz de instalar o LXC, criar e configurar contêineres CentOS 7, e gerenciar seu ciclo de vida.

## 1. Pré-requisitos

Antes de iniciar a instalação do LXC, é fundamental garantir que seu sistema CentOS 7 atenda a certos pré-requisitos. A instalação do LXC requer a adição do repositório EPEL (Extra Packages for Enterprise Linux) e a configuração de uma ponte de rede para a comunicação dos contêineres com a rede externa.

### 1.1 Adicionar o Repositório EPEL

O LXC não está disponível nos repositórios base do CentOS 7. Portanto, o primeiro passo é instalar o repositório EPEL, que fornece pacotes adicionais de alta qualidade para distribuições Linux baseadas em RHEL.

Para adicionar o repositório EPEL, execute o seguinte comando no seu terminal:

```bash
sudo yum -y install epel-release
```

Este comando instala o pacote `epel-release`, que configura o acesso ao repositório EPEL no seu sistema. A opção `-y` confirma automaticamente todas as solicitações de instalação.

### 1.2 Configurar uma Ponte de Rede (Network Bridge)

Os contêineres LXC necessitam de uma ponte de rede para se comunicarem com a rede externa e entre si. É crucial configurar uma ponte de rede antes de criar os contêineres. A documentação sugere que o nome da ponte de rede seja "virbr0" [1].

Para criar e configurar uma ponte de rede, você pode seguir os passos gerais para criação de pontes de rede no CentOS 7. Embora o artigo original mencione a necessidade de criar uma ponte de rede, ele não detalha os passos específicos. Uma abordagem comum envolve a criação de um arquivo de configuração de interface de rede para a ponte. Abaixo estão os passos para criar uma ponte chamada `virbr0`:

1.  **Criar o arquivo de configuração para a ponte `virbr0`:**

    Crie um novo arquivo em `/etc/sysconfig/network-scripts/ifcfg-virbr0` com o seguinte conteúdo:

    ```
    DEVICE=virbr0
    TYPE=Bridge
    ONBOOT=yes
    BOOTPROTO=static
    IPADDR=192.168.1.1
    NETMASK=255.255.255.0
    DELAY=0
    ```

    *   `DEVICE=virbr0`: Define o nome da interface da ponte.
    *   `TYPE=Bridge`: Especifica que esta é uma interface de ponte.
    *   `ONBOOT=yes`: Garante que a ponte seja ativada na inicialização do sistema.
    *   `BOOTPROTO=static`: Define um endereço IP estático para a ponte. Você pode ajustar `IPADDR` e `NETMASK` conforme sua rede local.
    *   `DELAY=0`: Define o atraso de ativação da ponte.

2.  **Modificar a interface de rede existente (ex: `ifcfg-eth0` ou `ifcfg-enpXsX`) para se juntar à ponte:**

    Edite o arquivo de configuração da sua interface de rede principal (por exemplo, `ifcfg-eth0` ou `ifcfg-enp0s3`). Remova as configurações de IP e adicione a linha `BRIDGE=virbr0`.

    Exemplo de `ifcfg-eth0` antes:

    ```
    TYPE=Ethernet
    BOOTPROTO=dhcp
    DEFROUTE=yes
    PEERDNS=yes
    PEERROUTES=yes
    IPV4_FAILURE_FATAL=no
    IPV6INIT=yes
    IPV6_AUTOCONF=yes
    IPV6_DEFROUTE=yes
    IPV6_PEERDNS=yes
    IPV6_PEERROUTES=yes
    IPV6_FAILURE_FATAL=no
    NAME=eth0
    UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    ONBOOT=yes
    ```

    Exemplo de `ifcfg-eth0` depois:

    ```
    TYPE=Ethernet
    BOOTPROTO=none
    DEFROUTE=yes
    PEERDNS=yes
    PEERROUTES=yes
    IPV4_FAILURE_FATAL=no
    IPV6INIT=yes
    IPV6_AUTOCONF=yes
    IPV6_DEFROUTE=yes
    IPV6_PEERDNS=yes
    IPV6_PEERROUTES=yes
    IPV6_FAILURE_FATAL=no
    NAME=eth0
    UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    ONBOOT=yes
    BRIDGE=virbr0
    ```

    *   `BOOTPROTO=none`: A interface não obterá um IP diretamente, pois será parte da ponte.
    *   `BRIDGE=virbr0`: Associa esta interface à ponte `virbr0`.

3.  **Reiniciar o serviço de rede:**

    Após as modificações, reinicie o serviço de rede para que as alterações entrem em vigor:

    ```bash
    sudo systemctl restart network
    ```

    Verifique se a ponte `virbr0` foi criada e está ativa com o comando `ip a` ou `brctl show`.

## 2. Instalação do LXC no CentOS 7

Com os pré-requisitos configurados, o próximo passo é instalar o LXC e os pacotes essenciais para o funcionamento dos contêineres.

### 2.1 Instalar Pacotes LXC

Execute o seguinte comando para instalar o LXC e suas dependências:

```bash
sudo yum -y install lxc lxc-templates libcap-devel libcgroup busybox wget bridge-utils lxc-extra
```

Este comando instala:

*   `lxc`: O pacote principal do Linux Containers.
*   `lxc-templates`: Modelos pré-definidos para a criação rápida de contêineres de diferentes distribuições.
*   `libcap-devel`: Bibliotecas de desenvolvimento para capacidades Linux.
*   `libcgroup`: Ferramentas para gerenciar cgroups (control groups), que são fundamentais para o isolamento de recursos dos contêineres.
*   `busybox`: Um conjunto de utilitários Unix simplificados, frequentemente usado em ambientes embarcados e contêineres.
*   `wget`: Utilitário para download de arquivos da web.
*   `bridge-utils`: Ferramentas para configurar pontes de rede.
*   `lxc-extra`: Pacotes adicionais que podem ser úteis para o LXC.

### 2.2 Verificar a Configuração do Kernel

Após a instalação, é recomendável verificar se o kernel do seu CentOS 7 está configurado corretamente para suportar o LXC. O utilitário `lxc-checkconfig` realiza essa verificação.

Execute o comando:

```bash
sudo lxc-checkconfig
```

Você deve ver uma saída indicando que as funcionalidades de namespaces, cgroups e outras configurações necessárias estão `enabled` (habilitadas). Se alguma configuração estiver desabilitada, pode ser necessário atualizar o kernel ou recompilá-lo com as opções apropriadas, embora isso seja raro em kernels modernos de CentOS 7.

## 3. Criação de Contêineres Linux

O LXC facilita a criação de contêineres usando modelos pré-configurados. Esses modelos automatizam o processo de instalação do sistema operacional dentro do contêiner.

### 3.1 Listar Modelos Disponíveis

Para ver quais modelos de contêiner estão disponíveis no seu sistema, execute:

```bash
ls /usr/share/lxc/templates/
```

Você verá uma lista de modelos como `lxc-centos`, `lxc-ubuntu`, `lxc-debian`, entre outros. Para este guia, focaremos no modelo `lxc-centos`.

### 3.2 Criar um Contêiner CentOS

Para criar um novo contêiner CentOS 7, utilize o comando `lxc-create`. Neste exemplo, criaremos um contêiner chamado `centos_lxc` usando o modelo `centos`.

```bash
sudo lxc-create -n centos_lxc -t centos
```

*   `-n <nome_do_contêiner>`: Especifica o nome do seu contêiner (neste caso, `centos_lxc`).
*   `-t <modelo>`: Indica qual modelo de sistema operacional usar (neste caso, `centos`).

Durante o processo de criação, o LXC fará o download dos arquivos necessários e configurará o sistema de arquivos base para o contêiner. Ao final, ele informará sobre a senha temporária do usuário `root` do contêiner, que geralmente é armazenada em `/var/lib/lxc/centos_lxc/tmp_root_pass`.

**Exemplo de Saída (parcial):**

```
Checking cache download in /var/cache/lxc/centos/x86_64/7/rootfs ...
Downloading centos minimal ...
...
Download complete.
Copy /var/cache/lxc/centos/x86_64/7/rootfs to /var/lib/lxc/centos_lxc/rootfs ...
Copying rootfs to /var/lib/lxc/centos_lxc/rootfs ...
...
Container rootfs and config have been created.
Edit the config file to check/enable networking setup.

The temporary root password is stored in:

        '/var/lib/lxc/centos_lxc/tmp_root_pass'

The root password is set up as expired and will require it to be changed
at first login, which you should do as soon as possible.
```

Anote a localização da senha temporária, pois você precisará dela para o primeiro login.

## 4. Credenciais e Acesso ao Contêiner

Após a criação, você precisará acessar o contêiner. O LXC fornece uma senha temporária para o usuário `root` e a opção de redefini-la.

### 4.1 Obter a Senha Temporária

A senha temporária do `root` do contêiner é armazenada em um arquivo dentro do diretório do contêiner no host. Para visualizá-la, execute:

```bash
sudo cat /var/lib/lxc/centos_lxc/tmp_root_pass
```

### 4.2 Redefinir a Senha do Root (Opcional, mas Recomendado)

Como a senha temporária expira e é gerada automaticamente, é altamente recomendável redefini-la para uma senha forte e de sua escolha. Você pode fazer isso diretamente do host, mesmo antes de iniciar o contêiner:

```bash
sudo chroot /var/lib/lxc/centos_lxc/rootfs passwd
```

Este comando permite que você defina uma nova senha para o usuário `root` dentro do sistema de arquivos do contêiner.

## 5. Iniciando e Acessando Contêineres Linux

Com o contêiner criado e as credenciais prontas, é hora de iniciá-lo e interagir com ele.

### 5.1 Iniciar o Contêiner

Para iniciar o contêiner em segundo plano (modo daemon), use o comando `lxc-start`:

```bash
sudo lxc-start -n centos_lxc -d
```

O contêiner `centos_lxc` agora estará em execução.

### 5.2 Acessar o Console do Contêiner

Para acessar o console do contêiner em execução, utilize `lxc-console`:

```bash
sudo lxc-console -n centos_lxc -t 0
```

*   `-n centos_lxc`: Especifica o nome do contêiner.
*   `-t 0`: Conecta-se ao `tty0` do contêiner. Em alguns casos, `tty1` pode não responder, então `tty0` é uma alternativa confiável.

Você será solicitado a inserir o nome de usuário (`root`) e a senha (a temporária ou a que você redefiniu). No primeiro login, o sistema pode forçar a alteração da senha, caso você não a tenha redefinido previamente.

Para sair do console do contêiner e retornar ao terminal do host, pressione `Ctrl+a` seguido de `q`.

Para reconectar-se a um contêiner já em execução, basta executar novamente o comando `lxc-console`.

## 6. Gerenciando Contêineres Linux

O LXC oferece comandos para listar, parar e destruir contêineres, permitindo um gerenciamento completo do ciclo de vida.

### 6.1 Listar Contêineres

Para listar todos os contêineres no host, execute:

```bash
sudo lxc-ls
```

Para ver informações mais detalhadas sobre os contêineres, incluindo seu estado (em execução, parado), IP e tipo de rede, use:

```bash
sudo lxc-ls -f
```

### 6.2 Parar um Contêiner

Para parar um contêiner em execução, utilize `lxc-stop`:

```bash
sudo lxc-stop -n centos_lxc
```

### 6.3 Destruir um Contêiner

Se você não precisar mais de um contêiner, pode destruí-lo completamente, o que removerá todos os seus arquivos e configurações:

```bash
sudo lxc-destroy -n centos_lxc
```

**Atenção:** Este comando é irreversível e removerá permanentemente o contêiner e todos os seus dados.

## 7. Configuração Avançada do Contêiner LXC

Cada contêiner LXC possui um arquivo de configuração individual que define seus parâmetros operacionais, como rede, recursos e dispositivos. Este arquivo é crucial para personalizar o comportamento do contêiner.

### 7.1 Localização do Arquivo de Configuração

O arquivo de configuração de cada contêiner está localizado em `/var/lib/lxc/<nome_do_contêiner>/config`. Para o nosso exemplo `centos_lxc`, o caminho seria:

```
/var/lib/lxc/centos_lxc/config
```

Você pode editar este arquivo usando um editor de texto como `vi` ou `nano`:

```bash
sudo vi /var/lib/lxc/centos_lxc/config
```

### 7.2 Exemplo de Configuração de Rede

Um dos aspectos mais importantes a configurar é a rede do contêiner. O exemplo abaixo mostra como configurar o contêiner para usar a ponte de rede `virbr0` que criamos anteriormente:

```ini
lxc.network.type = veth
lxc.network.link = virbr0
lxc.network.flags = up
lxc.network.hwaddr = 00:16:3e:XX:XX:XX
```

*   `lxc.network.type = veth`: Define o tipo de interface de rede como um par de Ethernet virtual. Isso cria duas interfaces virtuais, uma dentro do contêiner e outra no host, que são conectadas.
*   `lxc.network.link = virbr0`: Especifica que a interface do lado do host do par `veth` deve ser anexada à ponte de rede `virbr0`.
*   `lxc.network.flags = up`: Garante que a interface de rede esteja ativa quando o contêiner for iniciado.
*   `lxc.network.hwaddr = 00:16:3e:XX:XX:XX`: Define um endereço MAC exclusivo para a interface de rede do contêiner. É importante que este endereço seja único na sua rede para evitar conflitos. Os `XX:XX:XX` devem ser substituídos por valores aleatórios para garantir a unicidade.

Após fazer alterações no arquivo de configuração, você precisará reiniciar o contêiner para que as novas configurações sejam aplicadas.

## Conclusão

Este guia forneceu uma visão abrangente sobre a implantação e o gerenciamento de contêineres LXC no CentOS 7. Ao seguir estas etapas, você pode aproveitar a virtualização leve para isolar aplicações e serviços, otimizando o uso de recursos do seu servidor. O LXC é uma ferramenta poderosa para desenvolvedores e administradores de sistemas que buscam uma alternativa eficiente às máquinas virtuais tradicionais.

## Homepage
[https://informatizar.netlify.app/project/labs/backoffice/containers/centOS-LXC.html](https://informatizar.netlify.app/project/labs/backoffice/containers/centOS-LXC.html)

<br>
## Eduardo Schmidt

**`Informática para Internet`**

Atuando desde 1999 em Tecnologia da Informação, Eduardo Schmidt é um profissional experiente em suporte a soluções de automação comercial, terminais de transferência eletrônica, impressoras de cupons fiscais, sistemas operacionais e servidores da Microsoft e Linux. Plataformas de Hospedagens em Servidores Web. "[Eduardo.Inf.Br](https://informatizar.netlify.app/)".

---

<img 
    align="left" 
    alt="HTML"
    title="HTML" 
    width="30px" 
    style="padding-right: 10px;" 
    src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/html5/html5-original.svg" 
/>
<img 
    align="left" 
    alt="CSS" 
    title="CSS"
    width="30px" 
    style="padding-right: 10px;" 
    src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/css3/css3-original.svg" 
/>
<img 
    align="left" 
    alt="JavaScript" 
    title="JavaScript"
    width="30px" 
    style="padding-right: 10px;" 
    src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/javascript/javascript-original.svg" 
/>
<img 
    align="left" 
    alt="Bootstrap"
    title="Bootstrap" 
    width="30px" 
    style="padding-right: 10px;" 
    src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/bootstrap/bootstrap-original.svg" 
/>
<img 
    align="left" 
    alt="Git" 
    title="Git"
    width="30px" 
    style="padding-right: 10px;" 
    src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/git/git-original.svg" 
/>

<br/>

#### Participações | Conhecimentos

<p align="left">
  <img src="https://informatizar.netlify.app/id/img/logo-uolhost.png" alt="UOL Host" width="110"/>
  <img src="https://informatizar.netlify.app/id/img/logo-NCR.png" alt="NCR" width="110"/>
</p>
