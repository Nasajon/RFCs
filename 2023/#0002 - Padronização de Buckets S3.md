```
nsjRFC: 0002
Categoria: Padrão
Status: Rascunho
Substitui (nsjRFC): N.A.
Autores: Daniel Checchia
Data: Janeiro 2023
```

# **Boas Práticas para Buckets S3**

## **Resumo**

Este documento tem como objetivo apresentar as boas práticas de uso e segurança dos Buckets S3 e a padronização de uso por Produtos.

## **Status deste Documento**

Este documento é um Rascunho inicial. O objetivo é propor a discussão para que o uso dos Buckets tenham um padrão comum de uso e segurança para todos os produtos.

## **Aviso de direitos autorais**

Copyright (c) Todos os direitos relacionados são da Nasajon e seus Autores, nominados neste documento.

## **Introdução**

Nosso ambiente atual, devido ao grande crescimento e à falta de um padrão de organização, possui buckets fora dos padrões recomendados de segurança e de boas práticas.
Este documento tem como objetivos:
 - Padronizar tipos de buckets de acordo com o Produto ou área de negócios
 - Padronizar a criação de Buckets nos ambientes de Desenvolvimento, QA, Pilotos e Produção
 - Definir ciclo de vida dos dados nos buckets (até a sua exclusão)
 - Definir a estratégia de reorganização dos dados que estão nos buckets atuais.
Caberá à área de infraestrutura a criação e manutenção da nova estrutura, ficando para os desenvolvedores planejarem a migração dos dados.


## Tipificação de Buckets

Premissas iniciais de uso de buckets:
 - Por padrão, todos os buckets serão privados e acessíveis somente por meio de aplicações. usuários, clientes ou técnicos de suporte não terão acesso direto aos dados armazenados.
 - Todos os buckets serão nomeados com **nsjPRODUTO**
 - Cada produto possuirá seu próprio bucket e deve definir o ciclo de vida da informação.
 - O uso dos buckets será precificado e comporá o custo do produto dentro da infraestrutura
 - Dados comuns, compartilhados por vários produtos, devem ficar em um bucket próprio e seu custo será rateado por todos os produtos.
 - Qualquer outra situação deve ser aprovada pela equipe de infraestrutura.

Liberação de arquivos para downloads públicos de clientes:
 - As aplicações deverão gerar links temporários com, no máximo, 6 horas de validade.
 - Sempre que possível, a URL da AWS S3 deve ser "mascarada", usando a FQDN da Nasajon.

Repositório Público para download de backup de clientes cancelados:
 - A infraestrutura deverá criar um link dinâmico com duração de 30 dias;
 - O backup deve ficar armazenado no S3 por 90 dias;
 - Após 90 dias deve ser armazenado no S3 Glacier por mais 12 meses.
 - Expirado o tempo no Glacier, o arquivo deve ser excluído definitivamente.

Repositório Público área do Suporte de TI:
 - Repositório público para arquivos e ferramentas da TI
 - Não permitirá PUT, POST, DELETE - somente READ

Repositório CDN:

A infraestrutura está adotando um Bucket S3 como CDN, usando o Cloudfront e o domínio permanecerá o mesmo. As premissas serão:
 - No processo de compilação do executável, o arquivo subirá automaticamente para o Bucket de CDN.
 - Arquivos de versões muito antigas, com mais de 12 meses, serão movidos para o Glacier.
 - Após 12 meses no Glacier, o arquivo será excluído permanentemente.

## Possíveis abordagens de Migração de Dados


### Opção 1 - "On the Fly"
A idéia é incluir uma variável de ambiente com o Bucket Novo e, no código da aplicação, uma validação deve ser criada:
 - [ ] Verifica se está no bucket novo
 - [ ] Se não estiver no bucket novo, lê o arquivo no bucket antigo, salva no novo e remove do antigo

**Vantagens**
 - A alteração é direta no código, não dependendo de outros esforços.

**Desvantagens**
 - Pode gerar um tempo maior de resposta da aplicação, dependendo do tamanho do arquivo.
 - Nem todos arquivos são usados, o que deixará uma "sobra" de arquivos espalhada nos buckets
 - Não terá fim e nem um prazo final.

### Opção 2 - Movimentação Assíncrona
Nesta opção, o processo é assíncrono e não impacta na performance do aplicativo - porém demanda a criação de um worker para realizar as movimentações. Também seria criada uma nova variável de ambiente com o bucket novo. O processo ficaria da seguinte maneira:
 - [ ] Se for gravação do arquivo, já faz a gravação no bucket novo
 - [ ] Se o arquivo estiver no bucket antigo ele faz a leitura, cria uma msg para o rabbitMQ com a origem e destino
 - [ ] Se o arquivo não estiver no bucket antigo, ele buscará no bucket novo.
 - [ ] Um worker realizará a movimentação do bucket antigo para o novo.

**Vantagens**
 - Não causa impacto em performance
 - é um processo assíncrono para enfileirar a movimentação

**Desvantagens**
 - Nem todos arquivos são usados, o que deixará uma "sobra" de arquivos espalhada nos buckets.
 - Não terá fim e nem um prazo final.

### Opção 3 - Task Force de Produtos para Reorganização dos Dados
Esta opção é recomendada, mas onerosa. É uma ação que é mais rápida e garante a transferência de todos os arquivos, sem o risco de perda de dados quando a conta da AWS atual for removida.

Cada TechLeader seria o responsável pela movimentação dos dados, com o apoio técnico da área de infraestrutura - se necessário.

## Conclusão

Este processo de migração infelizmente precisa ocorrer. Caso haja outras maneiras, favor criar PRs para que seja analisada pelo **Conselho de Padronização.**
