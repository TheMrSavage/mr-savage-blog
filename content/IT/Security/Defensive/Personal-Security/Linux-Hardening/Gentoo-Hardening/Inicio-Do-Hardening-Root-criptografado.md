---
title: Inicio Do Hardening - Root criptografado
tags:
  - IT
  - Security
  - Defensive
  - Personal-Security
  - Linux-Hardening
  - Gentoo-Hardening
draft: false
date: 2025-06-20
---
# Motivação[^1]
 
Um pouco de contexto: Há um tempo (cerca de uns 2 ou 3 meses aproximadamente) finalizei a autobiografia de Edward Snowden, "Eterna Vigilância" , e, bem mais recentemente (cerca de 1 mês), finalizei o livro "Contagem Regressiva Até Zero day", de Kim Zetter (que é um livro que destrincha o caso Stuxnet com uma riqueza de detalhes impressionante). Nisso, somado a certas tendencias ideológicas pessoais, me levou a seguinte questão: "Como se proteger de uma ameaça a nível Estado?". Infelizmente ainda não possuo conhecimentos em Threat Modeling para listar eficientemente quais os limites de um APT (como o Equation Group ou Cozy Bears), então uma parte considerável das ideias foram surgindo de maneira "informal". No caso, a principal questão que me veio a mente foi: "Se você está lidando com uma ameaça a nível estado então você está lidando com uma ameaça que possui recursos praticamente ilimitados" - Ou seja, está lidando com um grupo que tem acesso a todos os dados relacionados a inteligencia de seu pais (supondo que seja um APT ligado ao Estado local), que possui acesso a diversos 0days (os quais você, a princípio, não tem como saber quais os vetores de ataque exatamente), e que possui recursos financeiros praticamente ilimitados. 

Lidar com uma ameaça desse nível é (evidentemente) extremamente desafiador, mas ainda é possível tomar algumas medidas para se proteger. Então, por meio de minhas pesquisas, surgiram diversas ideias. Irei apresentar neste texto a cadeia inicial, que consiste na [encriptação do sistema de arquivos excluindo a partição de boot](#^gentoo-wiki-rootfs-encryption), porque, como espero deixar claro ao decorrer do texto, é desnecessário a encriptação desta, e, na segunda parte, a [implementação do Secure Boot, no caso, do Secure Boot com geração de chaves próprias](#^gentoo-wiki-secure-boot).

Com relação a outras ideias (e medidas que não valem a pena serem detalhadas durante esse texto, por serem, honestamente, simples), temos a configuração do ssh para permitir o login de somente de usuários ["não-root"](#^disable-ssh-root-login-reference) e via [chaves](#^disable-ssh-password-auth) (em conjunto com a habilitação de um sistema de [logging](#^enable-ssh-loging) via [sysklogd](#^gentoo-wiki-sysklogd)), alteração da senha do usuário root para uma senha aleatória de cerca de 26 caracteres (e, seu respectivo armazenamento em uma entrada do keepass) e a habilitação de logging relacionado ao comando su. Algumas medidas que devo implementar posteriormente incluem a adoção do [2FA no ssh para logins com chave](#^enable-2fa-ssh), separação do header do LUKS em um dispositivo [separado](#^detached-luks-header) (para inutilizar completamente o sistema de arquivos sem o dispositivo físico), [firejail](#^gentoo-wiki-firejail) em toda aplicação que pode se apresentar como sendo um vetor de ataque e a implementação de firewall via [iptables](#^gentoo-wiki-iptables). 

Se você tem alguma sugestão de como tornar ainda mais "robusto" esse setup ficaria bastante feliz em ouvir pois, francamente, ando sem ideias de como tornar ele ainda mais eficiente.

Importante ressaltar que, durante o processo, como irá ficar claro ao decorrer da segunda parte, acabei trocando alguns aspectos na cadeia de boot. Sendo mais preciso, optei, por fins de simplicidade pela troca do GRUB para o boot direto via efibootmgr com UKI (Unified Kernel Image). Isto se deve a um problema que eu estava tendo onde, apesar de conseguir efetivamente assinar o kernel + módulos e o GRUB, não consegui assinar o initramfs. Sei que poderia embutir manualmente ele dentro do kernel e depois assinar, mas ai teria que re-configurar o GRUB para não buscar o arquivo do initramfs, e, na prática, o que eu estava fazendo era um UKI manual. Como não fazia sentido fazer isso manualmente toda vez (e utilizar um UKI faz a quantidade de arquivos necessários para boot cair pra somente um, o que deixa mais "clean" o sistema) e, como o GRUB nesse caso serviria somente para "redirecionar" o boot para o UKI, acabei optando por abandonar o GRUB mesmo. É uma linha a mais que eu preciso digitar no terminal, somente uma vez e até mesmo isso pode eventualmente ser automatizado.

# Primeira parte: Criptografia do disco

Certo, a primeira coisa a se fazer é ter um desenho mental relativo a cadeia de boot:

> Firmware ([BIOS](#^gentoo-wiki-bios)\/[UEFI](#^gentoo-wiki-uefi)) => [Bootloader](#^gentoo-wiki-bootloader) ([GRUB](#^gentoo-wiki-grub)\/UEFI) => [Kernel](#^gentoo-wiki-kernel) + [initramfs](#^gentoo-wiki-initramfs) => OS

Então, por partes, irei explicar, resumidamente, cada um dos componentes.

## Firmware

Não consegui achar nenhuma definição muito boa de firmware, mas é basicamente um software embutido no hardware (bare metal) necessário para o funcionamento do mesmo.

### BIOS

"Foi o padrão de firmware de computadores IBM-PC-Compativeis até ser descontinuado em 2020. Performa a inicialização do sistema, então carregando o setor de boot do dispositivo especificado na memoria [CMOS](#^gentoo-wiki-cmos) e o executando. Na maioria dos casos a secção de bot é a [MBR](#^gentoo-wiki-mbr)"

### UEFI

"UEFI (Unified Extensible Firmware Interface) é o padrão de firmware para bootar ROM feito para prover uma api estável para interação com o hardware do sistema. \[...\] A arquitetura UEFI define um sistema de inicialização diferente da BIOS. A principal diferença acontece após o sucesso da checagem propria e a inicialização do dispositivo. Internamente, [o UEFI então seleciona uma das entradas de boot](#^gentoo-wiki-efibootmgr) , determina a localização da aplicação EFI correspondente na partição de EFI (ESP) e inicializa. O processo é bem diferente da abordagem da BIOS, que simplesmente lê os primeiros 512 bytes do setor, como a MBR, na primeira lista de dispositivos disponíveis e executa." - Em suma, com UEFI tudo que você precisa é de uma aplicação EFI válida, em uma partição FAT (geralmente FAT32), e uma entrada que aponta para essa aplicação. O que simplifica bastante o processo de boot por eliminar a necessidade de um bootloader intermediário na MBR, e, com UKI, podendo reduzir os arquivos necessários para somente um.
## Bootloader

"É o programa responsável por carregar items bootaveis quando o sistema é inicializado"

### GRUB

"GRUB (ou, mais formalmente, GRUB2) é um bootloader multiboot capaz de carregar kernels de uma variedade de sistema de arquivos na maiora das arquiteturas de sistemas."

### UKI

"[UKI](#^gentoo-wiki-unified-kernel-image) (Unified Kernel Image) é um executavel único que pode ser bootado diretamente pelo EFI, ou automaticamente detectado pelo bootloader com pouca ou nenhuma configuração"

## Kernel

"O kernel é o núcleo do sistema operacional. Ele contêm a maior parte dos drivers de dispositivos, oferecendo interfaces para programas acessarem o hardware do sistema como memoria, GPU(s) e blocos de dispositivos"

## initramfs

"Um initramfs (**init**ial **ram f**ile **s**ystem) é usado para preparar o sistema linux durante o processo de boot, antes do processo de init começar. Ele normalmente se encarrega de montar sistemas de arquivos importantes (como \/usr ou \/var). **Usuarios que usam um sistema encriptado terão o initramfs pedindo pela senha antes do sistema de arquivos ser montado**"

## Preparação do disco

> Devo avisar antes de qualquer coisa que, esta secção corre um serio risco de ficar extremamente parecida com a própria wiki. Tentei deixar tão autoral quanto possível, porém, honestamente, não sei se atingi meu objetivo.

Todo o tutorial da Wiki pressupõe que está sendo feito durante a instalação base. Ainda tenho que criptografar tanto meu HD quanto o meu SSD no meu setup principal, então possivelmente (se for de interesse geral), devo relatar minha experiência em criptografar um Gentoo já instalado (apesar que, imagino, seja meramente questão de copiar os arquivos para outro lugar, reformatar as partições necessárias com LUKS, re-colocar os arquivos, re-compilar o kernel e re-criar as entradas de boot).

Bem, a Wiki sugere dois layouts:
1. Uma partição EFI com o bootloader + outros arquivos necessários para boot (como kernel ou o proprio initramfs), e uma partição com o sistema de arquivos de fato
2. Uma partição EFI com o bootloader, uma partição com os outros arquivos necessários para boot, e uma partição com o sistema de arquivos de fato

Para fins de simplicidade e praticidade (o que se mostrou um acerto quando migrei para o UKI), optei pelo layout mais simples.

Bem, o caminho é bem direto ao ponto, simplesmente crie duas partições (a wiki sugere usar o fdisk como ferramenta, que é o padrão, mas use a que achar melhor), uma com 1GiB (para ser a partição de EFI) e outra com o resto do tamanho disponível. 

### LUKS

["LUKS (Linux Unified Key Setup) é uma especificação de encriptação de disco"](#^wikipedia-luks)

Bem, para colocar o LUKS na partição será utilizado o dm-crypt, portanto é importante primeiro assegurar que o modulo do dm_crypt está habilitado. Isso pode ser feito com um simples modprobe e verificado usando lsmod. O restante do procedimento é simples, basta executar o seguinte comando:

> Lembrete: Como estamos fazendo a criptografia do rootfs somente, a "particao-desejada" deve ser o sistema de arquivos raiz/principal, não o EFI.


>```bash
>cryptsetup luksFormat /dev/particao-desejada
>``` 
>```text
>WARNING!
>=======
>This will overwrite data on /dev/particao-desejada irrevocably.
>Are you sure? (Type 'yes' in capital letters): 
>YES
>Enter passphrase for /dev/particao-desejada:
>```

Ai é só escolher a senha e pronto. LUKS instalado na partição. A wiki também recomenda fazer um backup do header. Isso pode ser feito com o seguinte comando:
> ```bash
> cryptsetup luksHeaderBackup /dev/particao-desejada --header-backup-file luks-headers.img
> ```

De resto, basta abrir o volume luks, que pode ser feito com o primeiro comando abaixo, formatar a partição aberta com o sistema de arquivos desejado (eu optei pelo btrfs), montar a partição já aberta e já configurada com o sistema de arquivos, e finalizar a instalação básica do Gentoo.

>```bash
>cryptsetup luksOpen /dev/particao-desejada root
>```

>```bash
>mkfs.btrfs -L rootfs /dev/mapper/root
>```

>```bash
>mount --label rootfs /mnt/gentoo
>```

## Configuração do initramfs

Bem, para a geração do initramfs a wiki recomenda utilizar o [dracut](#^gentoo-wiki-dracut) ou [ugrd](#^gentoo-wiki-ugrd). Eu optei pelo dracut, mas recomendo fortemente que você dê pelo menos uma olhada no ugrd (que, inclusive, foi em grande parte feito pela propria pessoa que escreveu as entradas na wiki relativas a encriptação do sistema de arquivos, tanto total quanto parcial).

Para fins de praticidade, optei por utilizar o [installkernel](#^gentoo-wiki-installkernel), sendo necessário compilar ele com as flags do dracut e grub (e, eventualmente, remove-la e troca-la pela flag do uki).

Com relação ao dracut, é importante acrescentar o modulo crypt em \/etc\/dracut.conf.d\/luks.conf, da seguinte forma:

> Importante: Mantenha os espaços entre as aspas e o modulo. Como esse está sendo acrescentado (e não sobrescrito) através do +=, a remoção do espaço pode causar confusão com a concatenação de palavras

>```text
>add_dracutmodules+=" crypt "
>``` 

Também é importante especificar o alvo (partição) para que o initramfs consiga descriptografar. Isso pode ser feito por meio do kernel_cmdline (e, caso você esteja usando o GRUB é necessário adicionar o parametro root também em GRUB_CMDLINE_LINUX_DEFAULT, dentro de \/etc\/default\/grub). Seguindo o mesmo exemplo da wiki:

> ```bash
>lsblk -o name,uuid
>NAME        UUID
>sdb                                           
>├─nvme0n1p1 BDF2-0139
>├─nvme0n1p2 b0e86bef-30f8-4e3b-ae35-3fa2c6ae705b
>└─nvme0n1p3 4bb45bd6-9ed9-44b3-b547-b411079f043b
>	└─root    cb070f9e-da0e-4bc5-825c-b01bb2707704
> ```

>```text
>kernel_cmdline+=" root=UUID=cb070f9e-da0e-4bc5-825c-b01bb2707704 rd.luks.uuid=4bb45bd6-9ed9-44b3-b547-b411079f043b "
>```

## Boot com o initramfs

### efibootmgr

Bem, basta criar a entrada da seguinte forma:

> Importante: Se, por algum motivo, a partição de ESP não for a primeira é necessário acrescentar o parametro --part numero-da-particao 

>```bash
>efibootmgr --create --disk /dev/partica-efi --label "Gentoo" --loader "vmlinuz-x.y.zz-gentoo" --unicode "initrd=initramfs-x.y.zz-gentoo"
>```
### GRUB

E, por fim, se o GRUB for utilizado basta utilizar o seguinte comando:

>```bash
>grub-install --efi-directory=/boot
>```

---
# Referencias:
1. https://www.digitalocean.com/community/tutorials/how-to-disable-root-login-on-ubuntu-20-04
	^disable-ssh-root-login-reference
2. https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server#step-4-disabling-password-authentication-on-your-server
	^disable-ssh-password-auth
3. https://www.simplified.guide/ssh/log-verbose
   ^enable-ssh-loging
4. https://wiki.gentoo.org/wiki/Sysklogd
   ^gentoo-wiki-sysklogd
5. https://www.digitalocean.com/community/tutorials/how-to-set-up-multi-factor-authentication-for-ssh-on-ubuntu-18-04
   ^enable-2fa-ssh
6. https://wiki.gentoo.org/wiki/Full_Disk_Encryption_From_Scratch#Detached_header
   ^detached-luks-header
7. https://wiki.gentoo.org/wiki/Firejail
   ^gentoo-wiki-firejail
8. https://wiki.gentoo.org/wiki/Iptables
   ^gentoo-wiki-iptables
9. https://wiki.gentoo.org/wiki/Rootfs_encryption
   ^gentoo-wiki-rootfs-encryption
10. https://wiki.gentoo.org/wiki/Secure_Boot
	^gentoo-wiki-secure-boot
11. https://wiki.gentoo.org/wiki/BIOS
	^gentoo-wiki-bios
12. https://wiki.gentoo.org/wiki/CMOS_BIOS_Memory
    ^gentoo-wiki-cmos
13. https://wiki.gentoo.org/wiki/Master_Boot_Record
	^gentoo-wiki-mbr
14. https://wiki.gentoo.org/wiki/Efibootmgr
	^gentoo-wiki-efibootmgr
15. https://wiki.gentoo.org/wiki/Bootloader
    ^gentoo-wiki-bootloader
16. https://wiki.gentoo.org/wiki/GRUB
    ^gentoo-wiki-grub
17. https://wiki.gentoo.org/wiki/Unified_kernel_image
	^gentoo-wiki-unified-kernel-image
18. https://wiki.gentoo.org/wiki/Kernel
	^gentoo-wiki-kernel
19. https://wiki.gentoo.org/wiki/Initramfs
    ^gentoo-wiki-initramfs
20. https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup
	^wikipedia-luks
21. https://wiki.gentoo.org/wiki/Dracut
	^gentoo-wiki-dracut
22. https://wiki.gentoo.org/wiki/UgRD
	^gentoo-wiki-ugrd
23. https://wiki.gentoo.org/wiki/Installkernel
    ^gentoo-wiki-installkernel


[^1]: Inicialmente, havia pensado em escrever esse artigo em inglês. Contudo, dado que, em minha opinião, as páginas da wiki já são suficientemente claras com relação a todos os passos que devem ser tomados (e não acho que tenha muito conteúdo técnico em português sobre), acabei reconsiderando e optei por escrever em português mesmo. Espero que o artigo seja de bom proveito a todos os usuários de Linux (apesar do texto ser direcionado para o Gentoo).