---
title: Minha Primeira Experiencia Com Borg Backup
tags:
  - IT
  - Home-Lab
  - Data-Hoarder
draft: false
date: 2025-08-03
---
# Contexto inicial

Nos últimos 3 meses, o meu interesse por *Data Hoarding* (literalmente, "acumulador/acumulo de dados")[^1] tem crescido consideravelmente. Isso por conta, muito provavelmente, de minhas leituras recentes, como o livro "Eterna Vigilância" de Edward Snowden, "Contagem Regressiva até Zero Day" de Kim Zetter, ou os textos dos criptoanarquistas/cypherpunks[^2]. Tais leituras reforçaram um ceticismo que já tinha com relação ao Estado e suas instituições, levando a um desejo ainda maior de autonomia.

Como uma coisa leva a outra, pesquisando sobre autonomia acabei caindo no subreddit [r/DataHoarder](https://www.reddit.com/r/DataHoarder/), focado no acumulo de dados e, consequentemente, como armazenar um volume grande de dados de maneira eficiente. Também cai no subreddit [r/homelab](https://www.reddit.com/r/homelab/) focado na construção de *homelabs*[^3].

Enfim, há algum tempo já havia notado a necessidade de backups, sobretudo locais. Porque, bem, você pode armazenar de maneira segura (usando criptografia) dados na nuvem, porém, além de ficar a mercê de um terceiro, dependendo do volume de dados (pense num cenário onde você deseja armazenar >1TB de informação), o custo pode ficar exponencialmente mais caro.

Como eu não gosto de manter dados potencialmente sensíveis expostos (e nem de gastar dinheiro atoa), procurei por uma solução. Encontrei-a no [subreddit de homelabs](https://www.reddit.com/r/homelab/wiki/faq/)

## Sobre backups

Bem, primeiro, **por que** ter um backup?

Há vários motivos para tal, mas todos se reduzem, de maneira ou de outra, a um: você *pode* (e provavelmente vai) perder os seus dados. Eis uma lista com alguns cenários:
- HDs/SSDs estragam com o tempo.
- Pode ocorrer uma falha de energia que frite sua placa mãe e os dispositions conectados via SATA.
- Você pode baixar um aplicativo suspeito e receber um ransomware de brinde.
- Sua casa pode ser inundada e todos os seus dispositivos eletrônicos estragam. 

Existem infinitas possibilidades e um bom backup pode te salvar. Agora, chegamos ao segundo ponto: *o que é um **bom** backup*? Bem, há duas perguntas implícitas aqui. Um bom backup sabe diferenciar *o que* deve ser armazenado e *como*.  

A pergunta "*o quê deve ser armazenado?*" é bastante pessoal. Existem diversas formas de categorizar dados e isso deve atender a sua demanda especifica[^4]. Sobre o "*como os dados devem ser armazenados?*", uma metodologia comum aplicada é a *3-2-1*, que consiste no seguinte:
- Três copias dos dados
- Dois dispositivos de media diferentes (como pendrives, hds externos, nuvem, cartões sd/microsd, etc)
- Um backup **fora** (*offsite*) de onde os dados originais estão.[^5]

Admito que, nessa minha primeira experiência, deixei a desejar ao somente possuir um backup via HD externo, estando ele no mesmo local que o HD principal (meu quarto). Pretendo fazer backup *offsite* e online (logicamente, criptografados) dos arquivos mais importantes (como as chaves que eu uso com o keepass e documentos), enquanto as minhas mídias (sobretudo de filmes, series, jogos) serão armazenadas em diversos cartões sd/microsd distintos. Mas isso são planos para o futuro.

Bem, agora chegamos a pergunta final: ***Qual** solução de backup utilizar?* De novo, aqui [não existe resposta certa](https://www.reddit.com/r/homelab/comments/1jcpm7g/what_backup_solution_are_you_using/). Existem diversas formas de se fazer um backup, desde as mais manuais ([como usar rsync + crontab](https://www.reddit.com/r/homelab/comments/1jcpm7g/comment/mi43mha/)) até as mais complexas ([como usar as ferramentas do sistema de arquivos *zfs*](https://docs.oracle.com/cd/E23824_01/html/821-1448/recover-3.html)).

Acabei optando pelo Borg. Motivo? Simplesmente me pareceu uma solução interessante, prática e eficiente para o problema.
## Sobre o Borg

Bem, como a [documentação](https://borgbackup.readthedocs.io/en/stable/index.html) define:

>**BorgBackup** (abreviado: **Borg**) é um programa de backup com deduplicação de dados. Opcionalmente, ele oferece suporte a compressão e criptografia autenticada.
>
>O principal objetivo do Borg é fornecer uma forma eficiente e segura de realizar backups. A técnica de deduplicação utilizada o torna adequado para backups diários, já que apenas as alterações são armazenadas. A criptografia autenticada torna o Borg apropriado mesmo para backups em destinos não totalmente confiáveis.

Em suma, o Borg enquanto solução permite a criptografia dos dados, [hospedagem *offsite*](https://borgbackup.readthedocs.io/en/stable/quickstart.html#remote-repositories), deduplicação (ou seja, elimina redundância de dados), [compressão](https://borgbackup.readthedocs.io/en/stable/quickstart.html#backup-compression), e [autenticação](https://borgbackup.readthedocs.io/en/stable/quickstart.html#passphrase-notes). É uma solução bastante robusta e confiável para backups tanto local quanto remotamente. 
# Seguindo o *Quickstart*

Bem, aqui vai ser mais um *copy-paste* da secção de *quickstart* da wiki. Infelizmente fiquei devendo algo mais aprofundado como uma analise do código fonte ou como exatamente o Borg garante todas as *features* listadas anteriormente. 

Meu objetivo, num primeiro momento, era simplesmente fazer o backup do meu HD principal para, além de garantir que eu não iria perder dados relevantes, conseguisse colocar criptografia via LUKS. Quem sabe, num futuro, não faça uma analise mais detalhada sobre essa solução.

## Criando um repositório

Para criar um repositório *local* no Borg basta utilizar o seguinte comando:

>```bash
>borg init --encryption=repokey /path/to/repo
>```

[Se desejar trocar o metodo de criptografia](https://borgbackup.readthedocs.io/en/stable/usage/init.html#more-encryption-modes).

## Criando uma entrada

Para criar uma entrada (pense como se fosse um commit), basta utilizar:

>```bash
>borg create /path/to/repo::archive-name /path1/ /path2/ ... /pathN/
>```

Recomendo sempre usar ```--verbose``` e ```--progress``` caso você esteja fazendo no terminal para ter uma noção melhor do que está sendo feito. E, caso esteja fazendo um próximo arquivo e queira ver mais informações (como compressão e tamanho da deduplicação), basta utilizar ```--stats```
## Listar todos os arquivos

>```bash
>borg list /path/to/repo
>```

## Extrair um arquivo

> OBS: Isso vai extrair o arquivo no path atual 

>```bash
>borg extract /path/to/repo::archive-name
>```

## Deletar um arquivo

> OBS: Ainda não testei essa opção (e nem a opção para recuperar o espaço em disco)

>```bash
>borg delete /path/to/repo::archive-name
>```

## Recuperar o espaço em disco

>```bash
>borg compact /path/to/repo
>```

# Conclusões

O processo de backup foi bem sucedido. A unica parte "estranha" (e, provavelmente por falha minha), foi o fato que, ao tentar extrair o backup para dentro de */mnt/hd* ele criou outra pasta também chamada de *mnt/hd* com os arquivos dentro.  Isso provavelmente ocorreu por ter gerado os arquivos com o path completo e ter extraído dentro do diretório e não em */mnt* (ou talvez no proprio */*?). 

Essa fato é meio irrelevante na prática dado que basta mover os arquivos de volta pro diretório e remover o diretório extra. E, honestamente, não acho que valha a pena arriscar extrair o backup para um diretório errado por algo tão simples.

De todo modo, devo ter mais uma experiencia em pouco tempo com o fluxo de backup[^6]. No mais lembre-se, como em criptos *not your keys, not your coins*. Não terceirize suas responsabilidades. Tome o controle daquilo que é importante para você.

[^1]: Eu acho que o termo *arquivista digital* seria melhor (e menos pejorativo) que *acumulador*, mas ok
[^2]: Aqui vale destacar os textos "[Porque eu escrevi o PGP](https://nakamotoinstitute.org/pt-br/library/porque-eu-escrevi-o-pgp/)", "[Libertaria in Cyberspace](https://www.activism.net/cypherpunk/libertaria.html)", "[The Crypto Anarchist Manifesto](https://www.activism.net/cypherpunk/crypto-anarchy.html)", "[A Cypherpunk's Manifesto](https://www.activism.net/cypherpunk/manifesto.html)", "[A Declaration of the Independence of Cyberspace](https://www.eff.org/cyberspace-independence)" e "[The Right to Read](https://www.gnu.org/philosophy/right-to-read.html)"
[^3]: [Homelab : a laboratory of (usually slightly outdated) awesome in the domicile](https://www.reddit.com/r/homelab/wiki/introduction/) 
[^4]: Eu, por exemplo, nessa primeira experiência optei por fazer backup total do meu HD, sem fazer uma separação clara. Apesar que acabei ignorando as minhas VMs por já desejar recria-las de outra forma.
[^5]: Isso salva naquele cenário da inundação que eu propus, por exemplo.
[^6]: No caso, ainda preciso criptografar (e colocar o *secureboot*) no sistema principal de arquivos de meu computador principal.
