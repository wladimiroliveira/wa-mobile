# Fundação do app mobile — Design

- **Data:** 2026-10-19
- **Repositório:** `wa-app`, árvore vazia, sem nenhum commit
- **API consumida:** `wa-api` em `main` (`52aa0e1`), prefixo `/v1`
- **Referência de funcionalidades:** `wa-web` em `main` (`744beba`)
- **Status:** aguardando revisão

## Problema

O `wa-system` é um sistema operacional de estoque e produção: catálogo,
fornecedores, receitas, entradas de estoque, lotes, perdas, pré-pesagens,
produções e controle de acesso. Hoje ele só tem cliente no navegador, o
`wa-web`. O chão de fábrica, que é quem mais registra informação, usa celular e
tablet, e precisa de um app nativo para Android e iOS a partir de um código
só.

O app tem que fazer tudo o que o `wa-web` faz. Tem que fazer também o que a API
já oferece e o web ainda não tem tela: usuários e produções. Em cima disso, ele
usa dois recursos que o navegador não entrega bem: leitura de código de barras
pela câmera e desbloqueio da sessão por biometria.

As exigências do `wa-web` continuam valendo, pelo mesmo motivo: o operador não
escolheu usar o sistema e não tolera sessão que cai ou tela que trava. A sessão
não pode cair sozinha. A tela nunca fica em branco sem explicação. O contrato
com a API tem que quebrar no compilador antes de quebrar em produção.

A restrição central do back end também é a mesma. Em `auth.service.ts`, um
refresh token já rotacionado que volta é lido como roubo, e a reação é revogar
**todas as sessões do usuário, em todos os dispositivos**. Um app que dispara
duas renovações ao mesmo tempo derruba o próprio operador, e derruba também a
sessão dele no web. Serializar a rotação é o requisito que organiza a
fundação.

## Restrições

- O `wa-api` é a **única** API. O app não guarda regra de negócio e nunca é a
  autoridade sobre permissão.
- **Sem offline.** É a mesma decisão do dono do produto registrada no `wa-web`:
  sem fila de escrita, sem banco local, sem sincronização. Rede caída é estado
  de erro com ação de repetir.
- Um código para Android e iOS.
- Distribuição dupla: interna (APK e TestFlight) **e** nas lojas (Google Play e
  App Store).
- Identificadores, arquivos, testes e comentários em inglês. Documentação e
  texto de tela em português.
- TDD sem exceção: teste vermelho primeiro, com a saída colada.

## Fatiamento

O app se divide em fatias independentes. Cada uma tem seu próprio PR, e as
fatias marcadas como *design próprio* ganham um documento antes do plano. Este
documento cobre a fatia 0.

| #   | Fatia            | Conteúdo                                                                                                              |
| --- | ---------------- | --------------------------------------------------------------------------------------------------------------------- |
| 0   | **Fundação**     | Projeto, cliente HTTP, sessão, biometria, login, troca de senha, navegação, controle por permissão, fornecedores      |
| 1   | Catálogo         | Produtos (busca, filtros, criar, editar, ativar, desativar, leitor de código de barras), receitas                     |
| 2   | Estoque          | Entrada (com leitor e o diálogo de saldo negativo), movimentações, análise de estoque, lotes por produto, perda por lote |
| 3   | Produção         | Pré-pesagens (lista, nova com plano do servidor, detalhe) e produções (*design próprio*: criar, listar por status, detalhe, finalizar, cancelar) |
| 4   | Administração    | Tipos de perda, perfis e permissões, usuários (*design próprio*: listar, criar, editar, redefinir senha)              |

A fatia 0 entrega fornecedores de ponta a ponta de propósito. É o menor CRUD da
API: só listar e criar. Ainda assim, ele passa por lista, formulário, permissão
de criação e invalidação de cache. Fundação sem consumidor real é fundação não
verificada.

Produções e usuários não têm tela no `wa-web` para servir de modelo. O desenho
dessas telas sai dos schemas da API e passa por revisão antes de virar código.

## Decisão 1 — Flutter

O app é escrito em Flutter (Dart). É um código só, compilado de forma nativa
para Android e iOS, com desempenho previsível em aparelho modesto, que é o caso
normal no chão de fábrica.

**Expo/React Native foi considerado e não escolhido** pelo dono do produto.
Ele permitiria reaproveitar parte do código do `wa-web`. A consequência aceita
está nos Riscos assumidos: regras de apresentação ficam duplicadas entre os dois
repositórios.

A versão do Flutter fica fixada no projeto. Atualizar a versão é uma mudança
explícita, com seu próprio commit.

## Decisão 2 — o contrato com a API é gerado, nunca escrito à mão

Nenhum modelo de requisição ou resposta do `wa-api` é digitado no app. Os
modelos Dart são gerados a partir de `../api/openapi.json`, o mesmo arquivo
versionado de que o `wa-web` gera seus tipos. Um campo que muda no back tem que
quebrar a análise estática do app, não a tela do operador.

O documento é OpenAPI 3.0.3, com 31 caminhos, uso pesado de `nullable` e poucos
`anyOf`. O gerador é escolhido na fatia 0 pelo resultado, não pela fama. Os
candidatos são `swagger_parser` (puro Dart, sem infraestrutura extra) e
`openapi-generator` com o alvo `dart-dio`. O critério de aceite:

- campo `nullable` vira tipo anulável em Dart;
- decimal continua `String` no modelo (ver Decisão 6);
- enum de permissão e de status vira enum Dart;
- o código gerado passa na análise estática sem exceções manuais.

O código gerado é versionado. Uma mudança de contrato aparece no diff, que é
onde ela deve ser discutida.

## Decisão 3 — sessão: token curto em memória, rotação serializada

### Onde cada token vive

O **access token** fica só em memória. O **refresh token** fica no
armazenamento seguro do sistema (`flutter_secure_storage`), que usa o Keychain
no iOS e o Keystore no Android. Ao abrir o app, o refresh token é usado para
reidratar a sessão.

Como no web, a reidratação roda a cada abertura. Assim o caminho de refresh é
exercitado o tempo todo e nunca vira um trecho que só falha no meio do turno.

### A renovação é serializada

Um app tem um processo só, então não existe o problema das várias abas do web.
Existe o mesmo problema dentro do processo: várias requisições paralelas
tomando `401` ao mesmo tempo.

O cliente HTTP (`dio`) faz **voo único**. A primeira resposta `401` dispara um
refresh. As demais requisições que falharem enquanto ele estiver em voo entram
numa fila, e todas são repetidas com o token novo quando ele chega. Quatro
chamadas paralelas produzem um refresh, não quatro.

Valem as duas regras do web:

- **O refresh token é lido do armazenamento dentro da seção serializada**,
  nunca capturado antes de esperar por ela.
- Quem entra na seção e encontra em memória um access token diferente do que
  falhou sabe que outra requisição já renovou, e reaproveita o token em vez de
  renovar de novo.

**Nem toda falha no refresh encerra a sessão.** Só o `wa-api` respondendo `401`
ou `403` à rotação significa "este token não é mais seu". Aí a sessão é limpa e
o usuário vai para o login, uma vez, sem laço. Um `502` ou uma falha de rede
deixam o refresh token onde está e viram nova tentativa.

### Ciclo de vida

- **Entrar:** `POST /v1/sessions/signin` guarda o par e busca `GET /v1/sessions/me`.
- **Reidratar:** ao abrir o app, e ao voltar do segundo plano com o access
  token vencido, faz refresh serializado. Sucesso segue para a tela pedida,
  falha vai para o login.
- **Sair:** `POST /v1/sessions/signout` com o refresh token. A limpeza local
  acontece mesmo se a chamada falhar: sair tem que sair.
- **Reinstalação no iOS:** o Keychain sobrevive à desinstalação do app. Na
  primeira abertura depois de instalar, o app apaga qualquer token herdado, para
  não reidratar a sessão de uma instalação anterior.

## Decisão 4 — biometria desbloqueia a sessão, não substitui a senha

A biometria é um portão local, opcional, sobre a sessão já existente. O
`wa-api` não sabe que ela existe.

- O primeiro login é sempre com usuário e senha.
- Depois do primeiro login, o app oferece ativar o desbloqueio por digital ou
  rosto (`local_auth`). A escolha fica guardada no armazenamento seguro.
- Com o desbloqueio ativo, o app exige a biometria ao abrir e ao voltar do
  segundo plano depois de um tempo de inatividade, **antes** de ler o refresh
  token. Sem biometria válida, nada sai do armazenamento.
- Falhar a biometria ou cancelar leva ao login com senha, sem apagar a sessão
  do servidor.
- Se o aparelho perder a biometria cadastrada, o desbloqueio é desativado e o
  usuário entra com senha.
- Trocar a senha, ou o servidor revogar os tokens, encerra a sessão. A
  biometria não ressuscita sessão que o servidor derrubou.

O tempo de inatividade que exige nova biometria é uma constante única na
fundação, de 5 minutos. O número muda quando o uso real pedir.

## Decisão 5 — a autorização do app é espelho, nunca autoridade

`GET /v1/sessions/me` devolve as permissões efetivas do usuário, já somadas
pelo back. O app usa essa lista para três coisas:

- o roteador (`go_router`) desvia para a tela de acesso negado quando falta a
  permissão exigida pela rota;
- a navegação só mostra os destinos permitidos;
- os botões de ação (criar, editar, registrar perda) só aparecem com a
  permissão correspondente.

A autoridade continua sendo o `wa-api`. **`403` é resposta legítima, não
defeito**: a interface a trata como estado previsto.

As permissões do back são as mesmas do web: `CATALOG_*`, `ENTRIES_*`,
`PRODUCTION_*`, `SALES_*`, `REPORTS_READ`, `LOSSES_*` e `ACCESS_*`. Os rótulos
em português e o agrupamento por área seguem `wa-web/src/lib/permissions.ts`.

## Decisão 6 — decimal nunca vira `double`

O `wa-api` trafega quantidade, peso e custo como texto
(`^\d{1,12}(\.\d{1,6})?$`), nunca como número JSON. O app mantém esse valor como
`String` no modelo gerado e converte para `Decimal` (pacote `decimal`) num
módulo só, na borda. `double` não entra em conta de dinheiro nem de peso, porque
erra o arredondamento.

A formatação para a tela usa `intl` em `pt_BR` (`R$ 1.234,56`, vírgula
decimal), com as mesmas regras de máscara do `wa-web/src/lib/decimal.ts`.

## Decisão 7 — o estado do servidor tem um dono só

Riverpod é o único guardião de tudo que veio do `wa-api`. Nada disso é copiado
para o estado local de um widget.

- Cada fatia declara seus providers, e cada mutação invalida exatamente o que
  sujou.
- As listas paginadas do back (`stock-movements`, `pre-weighings`,
  `productions`) usam o cursor da API (`limit`, `cursor`, `nextCursor`) com
  carregamento infinito.
- Voltar do segundo plano não revalida tudo. O padrão de revalidar ao ganhar
  foco vira, num aparelho que troca de app o tempo todo, uma rajada de
  requisições que o operador percebe como travamento.
- Estado de formulário e de interface é local à tela.

## Decisão 8 — recursos nativos

- **Código de barras:** leitura do GTIN pela câmera com `mobile_scanner`, nos
  formatos EAN-8, EAN-13, UPC-A e ITF-14. O código lido passa pela mesma
  validação de dígito verificador do `wa-web/src/lib/gtin.ts` antes de ser
  usado. A câmera é sempre alternativa à digitação, nunca o único caminho. A
  fundação não usa o leitor; ele entra nas fatias 1 e 2.
- **Biometria:** ver Decisão 4.
- **Permissões do sistema:** câmera e biometria pedem consentimento no
  primeiro uso, com texto em português que explica o motivo. Exigência das duas
  lojas.

## Estrutura do projeto

Fatias verticais, como no `wa-web`. Cada fatia é dona do seu acesso à API, dos
seus formulários, dos seus rótulos em português, das suas telas e dos seus
testes. Uma fatia nunca importa o interior de outra; o que duas precisam sobe
para `core` ou `ui`.

```
lib/
  app/            roteador, providers raiz, boot da sessão
  core/
    api/          modelos gerados do OpenAPI
    http/         cliente dio, ApiError, refresh serializado
    session/      tokens, sessão, biometria
    env/          configuração validada no boot
    format/       decimal, datas, unidades, GTIN
  ui/             tema, componentes, estados de carregando, erro e vazio
  features/
    auth/         login, troca de senha, acesso negado, rótulos de permissão
    suppliers/    listar e criar fornecedores
test/             espelha lib/
```

Arquivo grande é sinal de responsabilidade demais, e vira gatilho de divisão.

## Camada visual

Material 3 com tema próprio, na mesma paleta neutra do `wa-web` (primária quase
preta, destrutiva em vermelho, raio de 10px) e a fonte Geist. Tema escuro fica
fora desta fatia.

As tabelas do web viram **listas de cartões**. Por isso a personalização de
colunas do web (`/sessions/me/table-layouts`) não existe no app.

A navegação usa gaveta lateral, filtrada por permissão, no lugar da barra
lateral do web.

Alvo de toque de 48px e layout que funciona de celular pequeno a tablet são
requisito desde a primeira tela.

## Tratamento de erro

Um cliente HTTP só, e toda resposta fora da faixa 2xx vira um `ApiError` com
status e corpo já interpretado. O `wa-api` responde em dois formatos: os erros
de domínio vêm como `{message}`, e os de validação e o `429` vêm no formato
padrão do Fastify (`{statusCode, code, error, message}`). O `ApiError` aceita
os dois.

| Status             | Tratamento                                                                           |
| ------------------ | ------------------------------------------------------------------------------------ |
| 401                | Refresh serializado; falhando, limpa a sessão e vai ao login                         |
| 403                | Tela ou aviso de acesso negado; estado previsto, não erro                            |
| 404, 409           | Sobe para o campo do formulário que causou                                           |
| 429                | No login, avisa para esperar antes de tentar de novo                                 |
| 5xx, falha de rede | Nova tentativa com espera crescente; persistindo, estado de erro com ação de repetir |

**A mensagem da API não vai para a tela.** O `wa-api` responde em inglês e a
interface é em português. Cada fatia traduz status mais contexto para a frase
que o operador lê, como o `wa-web/src/lib/form-errors.ts` já faz.

Toda tela está sempre em um de três estados: carregando, com dado, ou com erro e
uma saída. Tela em branco e erro engolido em silêncio não são estados válidos.

## Ambiente e build

A base da API vem de `--dart-define=API_BASE_URL=...` e já inclui `/v1`. Ela é
lida e validada num módulo só, que falha alto no boot se faltar ou não tiver
esquema `http(s)`.

**HTTPS é requisito de produção.** Android (cleartext) e iOS (App Transport
Security) bloqueiam HTTP puro por padrão. Abrir exceção para HTTP só vale no
build de desenvolvimento, apontando para a API local.

| Comando                         | Papel                                          |
| ------------------------------- | ---------------------------------------------- |
| `flutter run`                   | Desenvolvimento num aparelho ou emulador       |
| `flutter test`                  | Suíte de testes                                |
| `flutter analyze`               | Análise estática, sem nenhum aviso aceito      |
| `dart format`                   | Formatação                                     |
| geração do contrato             | Gera os modelos a partir do `openapi.json`     |
| `flutter build apk`/`appbundle` | Build Android (APK interno, AAB para a Play)   |
| `flutter build ipa`             | Build iOS; **exige macOS**                     |

A análise estática roda com `strict-casts`, `strict-inference` e
`strict-raw-types` ligados.

Commitlint e hook de commit iguais aos dos outros repositórios, para a
convenção de commit valer aqui também.

## Testes

`flutter_test` e `mocktail`, com o HTTP simulado na borda do `dio`. Os dados
simulados são construídos **com os modelos gerados do OpenAPI**: um mock que
mente sobre o contrato não compila.

O teste de tela exercita comportamento pela interface: toca, digita e confere
o que aparece. Lógica pura (rótulo de permissão, conversão de decimal, validação
de GTIN, montagem de payload) é testada direto, sem montar widget.

Quatro coisas ganham teste próprio por serem o maior risco da fundação:

1. **Voo único do refresh:** várias requisições tomando `401` ao mesmo tempo
   resultam em **um** refresh, e todas são repetidas com o token novo.
2. **Leitura dentro da seção:** quem espera e encontra o token já renovado
   reaproveita o token e **não** dispara outro refresh.
3. **Guarda de permissão:** rota negada não monta a tela nem dispara a
   requisição.
4. **Portão de biometria:** sem biometria válida, o refresh token não é lido
   do armazenamento.

## Distribuição

| Canal       | Android                       | iOS                                                  |
| ----------- | ----------------------------- | ---------------------------------------------------- |
| Interno     | APK assinado, instalado direto | TestFlight                                           |
| Loja        | Google Play (AAB)             | App Store, via Custom App (Apple Business Manager) ou distribuição *unlisted* |

O que a distribuição exige, e não é código:

- conta Apple Developer (anual) e conta Google Play Console;
- **um ambiente macOS para o build iOS**, seja um Mac ou CI com macOS
  (Codemagic ou GitHub Actions). O desenvolvimento acontece no WSL, que não
  compila iOS;
- política de privacidade publicada, exigida pelas duas lojas;
- **uma conta de teste e uma API acessível pela internet, por HTTPS, para os
  revisores das lojas**. Um app atrás de login, com API só na rede local,
  é reprovado na revisão;
- chaves de assinatura Android e certificados iOS guardados fora do
  repositório.

A Apple costuma recusar na loja pública um app de uso interno de uma empresa.
Por isso o caminho previsto é o Custom App ou o *unlisted*, a confirmar com
quem administra as contas.

A configuração de CI e das lojas não faz parte da fatia 0. Ela é necessária
antes da primeira entrega.

## Escopo da entrega

**Fundação:**

- projeto Flutter com a versão fixada e análise estrita;
- cliente HTTP com `ApiError` e refresh de voo único com fila de repetição;
- sessão com token em memória, refresh token no armazenamento seguro e
  reidratação ao abrir e ao voltar do segundo plano;
- biometria opcional;
- guarda de sessão e de permissão no roteador;
- gaveta de navegação filtrada por permissão;
- tema e conjunto mínimo de componentes;
- modelos gerados do `openapi.json`;
- ambiente validado no boot;
- suíte de testes configurada.

**Telas:** login, boas-vindas, troca da própria senha, acesso negado, erro.

**Domínio:** fornecedores, com listar e criar (com `CATALOG_CREATE`).

**Fora do app:** nada. A fatia 0 não muda o `wa-api` nem o `wa-web`.

## Riscos assumidos

- **Revogação em cadeia é derrubada total.** O voo único resolve o app
  sozinho, e o app e o web não disputam token: cada login gera o seu par. Mas
  quando a revogação acontece, por defeito no voo único ou replay real, ela
  derruba o usuário em todos os clientes ao mesmo tempo, app e web. O teste de
  voo único é por isso o mais importante da fatia. Se um dia a reação do back
  se mostrar dura demais, o conserto é no `wa-api`, com janela de tolerância
  na rotação.
- **Duplicação entre app e web.** Máscara de decimal, rótulos de permissão,
  tradução de erro, regras de status de estoque e validação de GTIN existem em
  TypeScript no `wa-web` e passam a existir em Dart no app. Uma regra mudada
  num lado e esquecida no outro vira divergência visível ao operador. Mitigação:
  cada regra portada cita no código a origem no `wa-web`, e mudança de regra
  de apresentação toca os dois repositórios no mesmo ciclo.
- **A biometria é portão local.** Ela protege contra quem pega o aparelho
  desbloqueado. Não protege contra quem tem acesso ao aparelho com root ou
  jailbreak. A credencial continua sendo o refresh token no armazenamento
  seguro do sistema.
- **Sem offline.** Rede caída no chão de fábrica significa operador parado.
  Foi decisão explícita do dono do produto, tomada com o custo à vista.

## Fora de escopo

Offline e sincronização. Notificação push: o `wa-api` não tem registro de
aparelho nem envio. Tempo real: o `wa-api` não tem websocket nem SSE. Tema
escuro. Internacionalização. Personalização de colunas. Vendas: as permissões
`SALES_*` existem, mas o `wa-api` não tem rotas de vendas. Configuração de CI e
das lojas, que vem antes da primeira entrega e depois desta fatia.

## Achados no `wa-api`, relatados e não consertados

- **Formato de erro duplo.** Erros de domínio vêm como `{message}`, e erros de
  validação e o `429` vêm no formato padrão do Fastify. O próprio desenho de
  autenticação do back deixa isso para um tratador de erro futuro. O app
  aceita os dois.
- **Listas sem paginação.** `GET /v1/users`, `/roles`, `/suppliers`,
  `/products` e `/loss-reasons` devolvem array inteiro. Em celular, com rede
  móvel, o custo de uma lista grande aparece antes do que no desktop. Quando o
  volume de produtos crescer, a paginação precisa nascer no back.
- **Exposição da API não definida.** O `openapi.json` declara só
  `http://localhost:<porta>` como servidor, e nada no repositório diz onde a
  API roda em produção. A revisão das lojas exige uma API acessível pela
  internet, por HTTPS. Isso é decisão de infraestrutura, fora do código, e
  precisa estar resolvida antes da primeira submissão.
