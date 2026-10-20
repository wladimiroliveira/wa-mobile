# Fundação do app mobile — Implementation Plan

> **For agentic workers:** execute task by task, in order. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir a fundação do `wa-app` (projeto Flutter, contrato gerado, camada HTTP com renovação de voo único, sessão com portão de biometria, roteamento com guarda de permissão, casco e tema) e provar que ela funciona com fornecedores de ponta a ponta.

**Architecture:** App Flutter para Android e iOS consumindo o `wa-api` por HTTP. Os modelos e os clientes HTTP são gerados do `openapi.json` pelo `swagger_parser`. O estado de servidor pertence ao Riverpod. A sessão guarda o access token em memória e o refresh token no armazenamento seguro do sistema, e a rotação roda sob uma trava em processo para não disparar a revogação em cadeia do back end. A biometria é um portão local que impede a leitura do refresh token. As telas são fatias verticais que não conhecem o interior umas das outras.

**Tech Stack:** Flutter 3.47.7, Dart 3.13.5, flutter_riverpod 3.4.3, go_router 18.1.0, dio 5.11.1, retrofit 4.10.0, swagger_parser 1.45.0, json_serializable 6.15.1, build_runner 2.16.2, flutter_secure_storage 11.2.0, local_auth 3.0.2, shared_preferences 2.5.6, decimal 3.2.6, intl 0.20.3, mocktail 1.0.5. Versões resolvidas num projeto de teste em 2026-10-19; o `flutter pub add` da Task 1 pode resolver versões de correção mais novas, e o PR registra as que ficaram no `pubspec.lock`.

**Spec:** `docs/superpowers/specs/2026-10-19-mobile-app-foundation-design.md`

## Global Constraints

Valem para toda tarefa deste plano. Nenhuma tarefa pode afrouxá-las.

- **TDD sem exceção.** Teste que falha primeiro, com a saída do vermelho colada no relatório da tarefa, antes de qualquer linha de implementação. A única tarefa sem teste automatizado é a configuração nativa (Task 17), verificada por build e pela conferência manual, e isso fica dito no relatório dela.
- **Idioma.** Identificadores, arquivos, testes, comentários, mensagens de commit e nomes de branch em inglês. Português apenas no texto que o operador lê na tela.
- **Commits.** Uma linha, formato `type(scope): what the change does`, seguida só da linha de atribuição `Co-Authored-By`. Sem corpo.
- **Git.** Nunca trabalhar na `main`. Este plano roda na branch `feat/foundation`. Não commitar, dar push ou abrir PR sem ordem explícita do dono do projeto; os commits descritos nas tarefas só são executados quando ele autorizar.
- **Imports.** Sempre `package:wa_app/...`, nunca relativo que suba diretório.
- **O código gerado só é importado por `lib/core/api/api.dart`.** Nenhum outro arquivo de `lib/` ou `test/` importa `lib/core/api/generated/`. Nenhum nome numerado (`Product22`, `Items12`, `Supplier3`) aparece fora de `api.dart`.
- **A mensagem de erro da API nunca vai para a tela.** O `wa-api` responde em inglês; a interface é em português.
- **O app nunca é autoridade sobre permissão.** `403` é estado previsto, não defeito.
- **Decimal nunca vira `double`.** Esta fatia não manipula decimais, mas nenhum modelo gerado é alterado para mudar isso.
- **Tela pequena é caso previsto.** Todo alvo de toque tem ao menos 48px, e toda tela funciona de celular pequeno a tablet.
- **Análise estática sem aviso.** `flutter analyze` sai limpo em tudo fora de `lib/core/api/generated/`.
- **Um único repositório é tocado**: o `wa-app`. O `../api/openapi.json` é lido, nunca escrito.

## Decisões pendentes antes da Task 1

- **Identificador do app.** O plano usa `--org br.com.hipersenna --project-name wa_app`, que gera `br.com.hipersenna.wa_app` no Android e `br.com.hipersenna.waApp` no iOS. Esse identificador fica fixo nas lojas depois da primeira publicação. Confirmar com o dono do projeto antes de rodar a Task 1.

## Contrato do `wa-api` consumido nesta fatia

Base: `API_BASE_URL`, só a origem (ex.: `http://localhost:3333`). Os clientes gerados trazem o caminho completo.

| Rota                             | Permissão exigida | Corpo                              | Respostas                         |
| -------------------------------- | ----------------- | ---------------------------------- | --------------------------------- |
| `POST /v1/sessions/signin`       | pública           | `{ username, password }`           | `200` par de tokens, `401`, `429` |
| `POST /v1/sessions/refresh`      | pública           | `{ refreshToken }`                 | `200` par de tokens, `401`        |
| `POST /v1/sessions/signout`      | autenticado       | `{ refreshToken }`                 | `204`                             |
| `GET /v1/sessions/me`            | autenticado       | —                                  | `200` usuário e permissões        |
| `PATCH /v1/sessions/me/password` | autenticado       | `{ currentPassword, newPassword }` | `204`, `401` senha atual errada   |
| `GET /v1/suppliers`              | `CATALOG_READ`    | —                                  | `200` lista `{ id, name }`        |
| `POST /v1/suppliers`             | `CATALOG_CREATE`  | `{ name }` (1 a 120 caracteres)    | `201`, `409` nome repetido        |

Comportamentos do back que o app precisa respeitar:

- Refresh token já rotacionado que volta revoga **todas** as sessões do usuário.
- `PATCH /sessions/me/password` responde `401` para senha atual errada, com o mesmo status de token inválido. Um segundo `401`, depois de um refresh bem-sucedido, é a resposta do endpoint, não sessão morta. É o mesmo tratamento do `wa-web/src/lib/http.ts`.
- Trocar a senha revoga todos os refresh tokens do usuário.
- Erro de domínio vem como `{ message }`; validação e `429` vêm como `{ statusCode, code, error, message }`.

## File Structure

| Arquivo                                          | Responsabilidade                                                                  |
| ------------------------------------------------ | --------------------------------------------------------------------------------- |
| `lib/main.dart`                                  | Lê o ambiente, abre as preferências, monta o `ProviderScope`.                     |
| `lib/app/app.dart`                               | `WaApp`: `MaterialApp.router`, boot da sessão, ciclo de vida.                     |
| `lib/app/routes.dart`                            | Caminhos e a função pura `redirectFor`.                                           |
| `lib/app/router.dart`                            | `routerProvider`: tabela de rotas do `go_router`.                                 |
| `lib/app/app_shell.dart`                         | Casco: barra superior, gaveta filtrada por permissão, oferta de biometria.        |
| `lib/app/nav_items.dart`                         | Destinos da gaveta e a permissão de cada um.                                      |
| `lib/app/home_page.dart`                         | Boas-vindas.                                                                      |
| `lib/app/splash_page.dart`                       | Tela enquanto a sessão reidrata.                                                  |
| `lib/core/env/env.dart`                          | Lê e valida `API_BASE_URL`. Único lugar que toca `String.fromEnvironment`.        |
| `lib/core/api/generated/`                        | Gerado pelo `swagger_parser`. Nunca editado à mão.                                |
| `lib/core/api/api.dart`                          | Apelidos legíveis sobre o gerado. Único lugar que o importa.                      |
| `lib/core/api/api_providers.dart`                | `dioProvider` autenticado e `apiProvider`.                                        |
| `lib/core/http/api_error.dart`                   | `ApiError`, `SessionExpiredError`, `SessionLockedError`, `callApi`.               |
| `lib/core/http/serial_lock.dart`                 | Executa ações assíncronas uma de cada vez.                                        |
| `lib/core/http/auth_interceptor.dart`            | Cabeçalho `Authorization`, interceptação de `401`, uma repetição.                 |
| `lib/core/http/http_providers.dart`              | `envProvider`, adaptador substituível, `bareDioProvider`.                         |
| `lib/core/http/retry_policy.dart`                | Quando o Riverpod repete um provider que falhou.                                  |
| `lib/core/http/error_messages.dart`              | Traduz erro para a frase em português.                                            |
| `lib/core/session/secure_store.dart`             | Interface do armazenamento seguro e a implementação real.                         |
| `lib/core/session/app_prefs.dart`                | Preferências não secretas: instalação, sessão existente, biometria.               |
| `lib/core/session/fresh_install.dart`            | Apaga tokens herdados de uma instalação anterior.                                 |
| `lib/core/session/token_store.dart`              | Onde cada token vive, e a trava de sessão.                                        |
| `lib/core/session/token_refresher.dart`          | Renovação serializada, com leitura dentro da trava.                               |
| `lib/core/session/session_expiry.dart`           | Sinal síncrono de que o servidor encerrou a sessão.                               |
| `lib/core/session/biometric_auth.dart`           | Interface de biometria e a implementação com `local_auth`.                        |
| `lib/core/session/session_state.dart`            | Estados da sessão.                                                                |
| `lib/core/session/session_controller.dart`       | Boot, entrar, sair, travar, destravar, trocar senha.                              |
| `lib/core/session/session_providers.dart`        | Providers da sessão.                                                              |
| `lib/ui/theme.dart`                              | Tema Material 3 na paleta do `wa-web`, fonte Geist.                               |
| `lib/ui/async_value_view.dart`                   | Os três estados de toda tela: carregando, dado, erro com saída.                   |
| `lib/features/auth/login_page.dart`              | Entrada com usuário e senha.                                                      |
| `lib/features/auth/unlock_page.dart`             | Desbloqueio por biometria.                                                        |
| `lib/features/auth/unavailable_page.dart`        | Servidor fora do ar na abertura.                                                  |
| `lib/features/auth/change_password_page.dart`    | Troca da própria senha.                                                           |
| `lib/features/auth/access_denied_page.dart`      | Acesso negado.                                                                    |
| `lib/features/auth/require_permission.dart`      | Guarda que não monta a tela sem a permissão.                                      |
| `lib/features/suppliers/suppliers_repository.dart` | Acesso à API de fornecedores e o provider da lista.                             |
| `lib/features/suppliers/suppliers_page.dart`     | Lista de fornecedores.                                                            |
| `lib/features/suppliers/supplier_form_sheet.dart` | Cadastro de fornecedor.                                                          |
| `test/support/fake_adapter.dart`                 | Adaptador HTTP falso, roteado por método e caminho.                               |
| `test/support/fakes.dart`                        | `InMemorySecureStore`, `FakeBiometricAuth`.                                       |
| `test/support/samples.dart`                      | Amostras construídas com os modelos gerados.                                      |
| `test/support/harness.dart`                      | Monta o `ProviderContainer` de teste e o app.                                     |
| `scripts/dev.sh`                                 | `adb reverse` e `flutter run` com a API local.                                    |
| `scripts/generate-api.sh`                        | Regenera o contrato a partir do `openapi.json`.                                   |
| `scripts/check.sh`                               | Análise, formatação e testes.                                                     |

---

## Task 1: Esqueleto do projeto e ambiente validado

**Files:**

- Create: projeto Flutter na raiz do repositório
- Create: `lib/core/env/env.dart`
- Test: `test/core/env/env_test.dart`
- Create: `analysis_options.yaml`, `scripts/dev.sh`, `scripts/check.sh`
- Create: `package.json`, `commitlint.config.js`, `.husky/commit-msg`
- Modify: `.gitignore`, `pubspec.yaml`

**Interfaces:**

- Consumes: nada.
- Produces: de `package:wa_app/core/env/env.dart`, `Env` com `Uri apiOrigin`, `Env.parse(String raw)`, `Env.fromEnvironment()` e `EnvError`.

- [ ] **Step 1: Criar a branch e o projeto**

```bash
git switch main 2>/dev/null || true
git switch -c feat/foundation
flutter create --org br.com.hipersenna --project-name wa_app --platforms android,ios --empty .
flutter config --no-analytics
```

Apagar o `test/widget_test.dart` criado pelo template, se existir.

- [ ] **Step 2: Fixar a versão do Flutter e adicionar as dependências**

Em `pubspec.yaml`, na seção `environment`:

```yaml
environment:
  sdk: ^3.13.5
  flutter: 3.47.7
```

```bash
flutter pub add flutter_riverpod go_router dio retrofit json_annotation flutter_secure_storage local_auth shared_preferences decimal intl
flutter pub add -d swagger_parser build_runner json_serializable retrofit_generator mocktail flutter_lints
```

- [ ] **Step 3: Análise estrita**

`analysis_options.yaml`:

```yaml
include: package:flutter_lints/flutter.yaml

analyzer:
  exclude:
    - lib/core/api/generated/**
  language:
    strict-casts: true
    strict-inference: true
    strict-raw-types: true

linter:
  rules:
    - always_declare_return_types
    - avoid_dynamic_calls
    - prefer_single_quotes
    - unawaited_futures
```

- [ ] **Step 4: Escrever o teste que falha**

`test/core/env/env_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/env/env.dart';

void main() {
  group('Env.parse', () {
    test('accepts an http origin and keeps its port', () {
      final env = Env.parse('http://localhost:3333');

      expect(env.apiOrigin.toString(), 'http://localhost:3333');
    });

    test('accepts an https origin with a trailing slash', () {
      final env = Env.parse('https://api.example.com/');

      expect(env.apiOrigin.toString(), 'https://api.example.com');
    });

    test('fails loudly when the value is missing', () {
      expect(() => Env.parse(''), throwsA(isA<EnvError>()));
    });

    test('rejects a scheme other than http or https', () {
      expect(() => Env.parse('ftp://localhost:3333'), throwsA(isA<EnvError>()));
    });

    test('rejects a value without a host', () {
      expect(() => Env.parse('localhost'), throwsA(isA<EnvError>()));
    });

    test('rejects a path, because the generated clients already add /v1', () {
      expect(() => Env.parse('http://localhost:3333/v1'), throwsA(isA<EnvError>()));
    });

    test('rejects a query string', () {
      expect(() => Env.parse('http://localhost:3333?x=1'), throwsA(isA<EnvError>()));
    });
  });
}
```

- [ ] **Step 5: Rodar o teste e confirmar o vermelho**

Run: `flutter test test/core/env/env_test.dart`
Expected: FAIL, porque `package:wa_app/core/env/env.dart` não existe.

- [ ] **Step 6: Escrever a implementação mínima**

`lib/core/env/env.dart`:

```dart
/// Configuration read once at boot. Nothing else in the app reads
/// `String.fromEnvironment`, so a missing or malformed value fails here, loudly,
/// instead of at the first request.
final class Env {
  const Env({required this.apiOrigin});

  final Uri apiOrigin;

  factory Env.fromEnvironment() =>
      Env.parse(const String.fromEnvironment('API_BASE_URL'));

  factory Env.parse(String raw) {
    if (raw.isEmpty) {
      throw const EnvError(
        'API_BASE_URL is missing; pass --dart-define=API_BASE_URL=http://host:port',
      );
    }
    final uri = Uri.tryParse(raw);
    if (uri == null ||
        !(uri.isScheme('http') || uri.isScheme('https')) ||
        uri.host.isEmpty) {
      throw EnvError('API_BASE_URL must be an http(s) origin, got "$raw"');
    }
    // The generated clients carry the full path, /v1 included. A base url with
    // a path would double it, which is the habit the web app's config teaches.
    if (uri.path.isNotEmpty && uri.path != '/') {
      throw EnvError('API_BASE_URL must be an origin without a path, got "$raw"');
    }
    if (uri.hasQuery || uri.hasFragment) {
      throw EnvError('API_BASE_URL must not carry a query or fragment, got "$raw"');
    }
    return Env(
      apiOrigin: Uri(
        scheme: uri.scheme,
        host: uri.host,
        port: uri.hasPort ? uri.port : null,
      ),
    );
  }
}

final class EnvError implements Exception {
  const EnvError(this.message);

  final String message;

  @override
  String toString() => 'EnvError: $message';
}
```

- [ ] **Step 7: Rodar o teste e confirmar o verde**

Run: `flutter test test/core/env/env_test.dart`
Expected: PASS, 7 testes.

- [ ] **Step 8: Scripts de desenvolvimento e verificação**

`scripts/dev.sh`:

```bash
#!/usr/bin/env bash
# Runs the app on the USB-connected device against the API on this machine.
# `adb reverse` makes the phone's localhost reach this machine over the cable,
# so no Wi-Fi is needed. It has to be redone every time the phone reconnects.
set -euo pipefail

port="${API_PORT:-3333}"
adb reverse "tcp:${port}" "tcp:${port}"
exec flutter run --dart-define=API_BASE_URL="http://localhost:${port}" "$@"
```

`scripts/check.sh`:

```bash
#!/usr/bin/env bash
# The project gate: static analysis, formatting and the whole test suite.
set -euo pipefail

flutter analyze
git ls-files -co --exclude-standard '*.dart' | grep -v '^lib/core/api/generated/' \
  | xargs dart format --output=none --set-exit-if-changed
flutter test
```

```bash
chmod +x scripts/dev.sh scripts/check.sh
```

- [ ] **Step 9: Convenção de commit igual à dos outros repositórios**

`package.json`:

```json
{
  "name": "wa-app-tooling",
  "private": true,
  "scripts": {
    "prepare": "husky"
  }
}
```

`commitlint.config.js`:

```js
export default { extends: ["@commitlint/config-conventional"] };
```

```bash
npm install -D husky @commitlint/cli @commitlint/config-conventional
npx husky init
echo 'npx --no -- commitlint --edit "$1"' > .husky/commit-msg
rm -f .husky/pre-commit
```

Acrescentar ao `package.json` a linha `"type": "module"`, exigida pelo `export default` do `commitlint.config.js`. Acrescentar `node_modules/` ao `.gitignore`.

- [ ] **Step 10: Verificar**

Run: `flutter analyze && flutter test`
Expected: `No issues found!` e 7 testes verdes.

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "chore(app): scaffold the Flutter project with a validated environment" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 2: Contrato gerado do OpenAPI

**Files:**

- Create: `swagger_parser.yaml`, `scripts/generate-api.sh`
- Create: `lib/core/api/generated/` (gerado)
- Create: `lib/core/api/api.dart`
- Create: `test/support/samples.dart`
- Test: `test/core/api/api_test.dart`

**Interfaces:**

- Consumes: `../api/openapi.json`.
- Produces: de `package:wa_app/core/api/api.dart`, os clientes `WaApi`, `AuthClient`, `SuppliersClient`; os apelidos `CurrentUser`, `Permission`, `TokenPair`, `RefreshedTokenPair`, `SignInBody`, `RefreshBody`, `SignOutBody`, `ChangePasswordBody`, `Supplier`, `CreateSupplierBody`. De `test/support/samples.dart`, `aCurrentUser(...)`, `aTokenPair(...)`, `aSupplier(...)`.

- [ ] **Step 1: Configurar e rodar a geração**

`swagger_parser.yaml`:

```yaml
swagger_parser:
  schema_path: ../api/openapi.json
  output_directory: lib/core/api/generated
  json_serializer: json_serializable
  root_client: true
  root_client_name: WaApi
```

`scripts/generate-api.sh`:

```bash
#!/usr/bin/env bash
# Regenerates the API contract from the wa-api OpenAPI document. The output is
# versioned: a contract change shows up in the diff, where it gets discussed.
set -euo pipefail

dart run swagger_parser
dart run build_runner build --delete-conflicting-outputs
```

```bash
chmod +x scripts/generate-api.sh
scripts/generate-api.sh
```

Expected: `13 rest clients, 42 requests` na saída do `swagger_parser`, e o `build_runner` termina sem erro.

- [ ] **Step 2: Escrever o teste que falha**

O teste prova que os apelidos apontam para os tipos certos, decodificando o JSON que a API realmente devolve.

`test/core/api/api_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/api/api.dart';

void main() {
  test('CurrentUser decodes /sessions/me, permissions included', () {
    final user = CurrentUser.fromJson({
      'id': '0199a8a0-0000-7000-8000-000000000001',
      'name': 'Ana',
      'username': 'ana',
      'roleId': null,
      'permissions': ['CATALOG_READ', 'CATALOG_CREATE'],
    });

    expect(user.name, 'Ana');
    expect(user.roleId, isNull);
    expect(user.permissions, [Permission.catalogRead, Permission.catalogCreate]);
  });

  test('an unknown permission from a newer API does not break decoding', () {
    final user = CurrentUser.fromJson({
      'id': '0199a8a0-0000-7000-8000-000000000001',
      'name': 'Ana',
      'username': 'ana',
      'roleId': null,
      'permissions': ['SOMETHING_NEW'],
    });

    expect(user.permissions, [Permission.$unknown]);
  });

  test('TokenPair and RefreshedTokenPair decode the same shape', () {
    final json = {'accessToken': 'a', 'refreshToken': 'r'};

    expect(TokenPair.fromJson(json).accessToken, 'a');
    expect(RefreshedTokenPair.fromJson(json).refreshToken, 'r');
  });

  test('Supplier decodes an item of GET /suppliers', () {
    final supplier = Supplier.fromJson({
      'id': '0199a8a0-0000-7000-8000-000000000002',
      'name': 'Moinho Santa Clara',
    });

    expect(supplier.name, 'Moinho Santa Clara');
  });

  test('request bodies encode the field names the API expects', () {
    expect(const SignInBody(username: 'ana', password: 'x').toJson(),
        {'username': 'ana', 'password': 'x'});
    expect(const RefreshBody(refreshToken: 'r').toJson(), {'refreshToken': 'r'});
    expect(const SignOutBody(refreshToken: 'r').toJson(), {'refreshToken': 'r'});
    expect(
        const ChangePasswordBody(currentPassword: 'a', newPassword: 'b').toJson(),
        {'currentPassword': 'a', 'newPassword': 'b'});
    expect(const CreateSupplierBody(name: 'X').toJson(), {'name': 'X'});
  });
}
```

- [ ] **Step 3: Rodar o teste e confirmar o vermelho**

Run: `flutter test test/core/api/api_test.dart`
Expected: FAIL, porque `package:wa_app/core/api/api.dart` não existe.

- [ ] **Step 4: Escrever os apelidos**

`lib/core/api/api.dart`:

```dart
// The only file that imports the generated contract. The generator names
// nested inline objects with numbers (Product22, Items12) that shift whenever
// the API grows, so feature code never names them: it uses the aliases below,
// built on the per-route names, which are stable. When a nested type needs a
// name, its alias goes here too, and a renumbering breaks the build in this one
// place.
import 'generated/export.dart';

export 'generated/export.dart'
    show
        AuthClient,
        GetV1SessionsMeResponse,
        GetV1SuppliersResponse,
        Permissions,
        PostV1SessionsRefreshResponse,
        PostV1SessionsSigninResponse,
        SuppliersClient,
        V1SessionsMePasswordRequestBody,
        V1SessionsRefreshRequestBody,
        V1SessionsSigninRequestBody,
        V1SessionsSignoutRequestBody,
        V1SuppliersRequestBody,
        WaApi;

typedef CurrentUser = GetV1SessionsMeResponse;
typedef Permission = Permissions;
typedef TokenPair = PostV1SessionsSigninResponse;
typedef RefreshedTokenPair = PostV1SessionsRefreshResponse;
typedef SignInBody = V1SessionsSigninRequestBody;
typedef RefreshBody = V1SessionsRefreshRequestBody;
typedef SignOutBody = V1SessionsSignoutRequestBody;
typedef ChangePasswordBody = V1SessionsMePasswordRequestBody;
typedef Supplier = GetV1SuppliersResponse;
typedef CreateSupplierBody = V1SuppliersRequestBody;
```

O `show` mantém os nomes numerados fora do resto do app. Se algum construtor gerado não for `const`, retirar o `const` do teste, nunca editar o gerado.

- [ ] **Step 5: Amostras tipadas pelo contrato**

`test/support/samples.dart`:

```dart
import 'package:wa_app/core/api/api.dart';

// Built from the generated models, so a sample that lies about the contract
// does not compile.

CurrentUser aCurrentUser({
  String name = 'Ana',
  List<Permission> permissions = const [],
}) =>
    CurrentUser(
      id: '0199a8a0-0000-7000-8000-000000000001',
      name: name,
      username: 'ana',
      roleId: null,
      permissions: permissions,
    );

TokenPair aTokenPair({String access = 'access-1', String refresh = 'refresh-1'}) =>
    TokenPair(accessToken: access, refreshToken: refresh);

Supplier aSupplier({String name = 'Moinho Santa Clara', String? id}) => Supplier(
      id: id ?? '0199a8a0-0000-7000-8000-000000000002',
      name: name,
    );
```

- [ ] **Step 6: Rodar o teste e confirmar o verde**

Run: `flutter test test/core/api/api_test.dart && flutter analyze`
Expected: PASS, 5 testes; `No issues found!`.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(api): generate the API contract from the wa-api OpenAPI document" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 3: Erro de API e trava serial

**Files:**

- Create: `lib/core/http/api_error.dart`, `lib/core/http/serial_lock.dart`
- Test: `test/core/http/api_error_test.dart`, `test/core/http/serial_lock_test.dart`

**Interfaces:**

- Consumes: `dio`.
- Produces: `ApiError(int status, {String? message, String? code})`, `ApiError.fromDio(DioException)`, `SessionExpiredError`, `SessionLockedError`, `Future<T> callApi<T>(Future<T> Function())`; `SerialLock` com `Future<T> run<T>(Future<T> Function())`.

- [ ] **Step 1: Escrever os testes que falham**

`test/core/http/api_error_test.dart`:

```dart
import 'package:dio/dio.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/http/api_error.dart';

DioException _withResponse(int status, Object? data) {
  final request = RequestOptions(path: '/v1/x');
  return DioException.badResponse(
    statusCode: status,
    requestOptions: request,
    response: Response<Object?>(requestOptions: request, statusCode: status, data: data),
  );
}

void main() {
  test('reads status and message from a domain error body', () {
    final error = ApiError.fromDio(_withResponse(409, {'message': 'Duplicate.'}));

    expect(error.status, 409);
    expect(error.message, 'Duplicate.');
  });

  test('reads code from the default Fastify error body', () {
    final error = ApiError.fromDio(_withResponse(429, {
      'statusCode': 429,
      'code': 'FST_ERR_RATE_LIMIT',
      'error': 'Too Many Requests',
      'message': 'Rate limit exceeded',
    }));

    expect(error.status, 429);
    expect(error.code, 'FST_ERR_RATE_LIMIT');
  });

  test('tolerates a body that is not a json object', () {
    final error = ApiError.fromDio(_withResponse(502, '<html>Bad Gateway</html>'));

    expect(error.status, 502);
    expect(error.message, isNull);
  });

  test('a failure with no response is status 0, the server was unreachable', () {
    final error = ApiError.fromDio(DioException.connectionError(
      requestOptions: RequestOptions(path: '/v1/x'),
      reason: 'refused',
    ));

    expect(error.status, 0);
  });

  test('an ApiError carried inside a DioException comes out as itself', () {
    final error = ApiError.fromDio(DioException(
      requestOptions: RequestOptions(path: '/v1/x'),
      error: const SessionExpiredError(),
    ));

    expect(error, isA<SessionExpiredError>());
  });

  test('callApi turns a DioException into an ApiError', () async {
    await expectLater(
      callApi<void>(() async => throw _withResponse(404, {'message': 'Not found.'})),
      throwsA(isA<ApiError>().having((e) => e.status, 'status', 404)),
    );
  });

  test('callApi returns the value when the call succeeds', () async {
    expect(await callApi(() async => 42), 42);
  });
}
```

`test/core/http/serial_lock_test.dart`:

```dart
import 'dart:async';

import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/http/serial_lock.dart';

void main() {
  test('runs actions one at a time, in arrival order', () async {
    final lock = SerialLock();
    final log = <String>[];
    final firstCanFinish = Completer<void>();

    final first = lock.run(() async {
      log.add('first:start');
      await firstCanFinish.future;
      log.add('first:end');
    });
    final second = lock.run(() async => log.add('second'));

    await Future<void>.delayed(Duration.zero);
    expect(log, ['first:start']);

    firstCanFinish.complete();
    await Future.wait([first, second]);
    expect(log, ['first:start', 'first:end', 'second']);
  });

  test('a failing action does not jam the ones queued behind it', () async {
    final lock = SerialLock();

    final failing = lock.run<void>(() async => throw StateError('boom'));
    final next = lock.run(() async => 'ran');

    await expectLater(failing, throwsStateError);
    expect(await next, 'ran');
  });
}
```

- [ ] **Step 2: Rodar os testes e confirmar o vermelho**

Run: `flutter test test/core/http`
Expected: FAIL, porque os dois módulos não existem.

- [ ] **Step 3: Escrever a implementação mínima**

`lib/core/http/api_error.dart`:

```dart
import 'package:dio/dio.dart';

/// Any answer outside 2xx, already interpreted. Status 0 means the server was
/// never reached. The API answers in English, so [message] is for logs and
/// branching, never for the screen.
class ApiError implements Exception {
  const ApiError(this.status, {this.message, this.code});

  factory ApiError.fromDio(DioException exception) {
    final carried = exception.error;
    if (carried is ApiError) return carried;

    final response = exception.response;
    if (response == null) return const ApiError(0);

    final data = response.data;
    final body = data is Map<String, Object?> ? data : null;
    return ApiError(
      response.statusCode ?? 0,
      message: body?['message'] as String?,
      code: body?['code'] as String?,
    );
  }

  final int status;
  final String? message;
  final String? code;

  @override
  String toString() => 'ApiError($status, $message)';
}

/// The API refused the refresh token: the session is over on the server.
class SessionExpiredError extends ApiError {
  const SessionExpiredError() : super(401);
}

/// The biometric gate is closed, so the refresh token must not be read.
class SessionLockedError extends ApiError {
  const SessionLockedError() : super(401);
}

Future<T> callApi<T>(Future<T> Function() call) async {
  try {
    return await call();
  } on DioException catch (exception) {
    throw ApiError.fromDio(exception);
  }
}
```

`lib/core/http/serial_lock.dart`:

```dart
/// Runs asynchronous actions one at a time, in the order they arrive. An app is
/// a single process, so this is all the token rotation needs to never run
/// twice at once.
class SerialLock {
  Future<void> _tail = Future<void>.value();

  Future<T> run<T>(Future<T> Function() action) {
    final result = _tail.then((_) => action());
    _tail = result.then<void>((_) {}, onError: (Object _) {});
    return result;
  }
}
```

- [ ] **Step 4: Rodar os testes e confirmar o verde**

Run: `flutter test test/core/http`
Expected: PASS, 9 testes.

- [ ] **Step 5: Commit**

```bash
git add lib/core/http test/core/http
git commit -m "feat(http): interpret API failures and serialize async actions" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 4: Armazenamento seguro, preferências e instalação nova

**Files:**

- Create: `lib/core/session/secure_store.dart`, `lib/core/session/app_prefs.dart`, `lib/core/session/fresh_install.dart`, `lib/core/session/token_store.dart`
- Create: `test/support/fakes.dart`
- Test: `test/core/session/token_store_test.dart`, `test/core/session/fresh_install_test.dart`

**Interfaces:**

- Consumes: `flutter_secure_storage`, `shared_preferences`.
- Produces:
  - `SecureStore` (interface: `read`, `write`, `delete`, `deleteAll`) e `FlutterSecureStore`.
  - `AppPrefs(SharedPreferences)` com `installed`/`markInstalled()`, `hasSession`/`setHasSession(bool)`, `biometricEnabled`/`setBiometricEnabled(bool)`, `biometricOffered`/`markBiometricOffered()`.
  - `Future<void> wipeOnFreshInstall(AppPrefs, SecureStore)`.
  - `TokenStore(SecureStore)` com `String? accessToken`, `bool locked`, `readRefreshToken()`, `save({access, refresh})`, `clear()`, `lock()`, `unlock()`, e a constante `TokenStore.refreshTokenKey`.
  - Em `test/support/fakes.dart`: `InMemorySecureStore` (com a lista `reads`) e `FakeBiometricAuth` (este, completado na Task 7).

- [ ] **Step 1: Escrever os testes que falham**

`test/support/fakes.dart`:

```dart
import 'package:wa_app/core/session/secure_store.dart';

class InMemorySecureStore implements SecureStore {
  final Map<String, String> values = {};

  /// Every key read, in order. Lets a test prove a key was never read.
  final List<String> reads = [];

  @override
  Future<String?> read(String key) async {
    reads.add(key);
    return values[key];
  }

  @override
  Future<void> write(String key, String value) async => values[key] = value;

  @override
  Future<void> delete(String key) async => values.remove(key);

  @override
  Future<void> deleteAll() async => values.clear();
}
```

`test/core/session/token_store_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/session/token_store.dart';

import '../../support/fakes.dart';

void main() {
  late InMemorySecureStore secure;
  late TokenStore tokens;

  setUp(() {
    secure = InMemorySecureStore();
    tokens = TokenStore(secure);
  });

  test('starts with no session at all', () async {
    expect(tokens.accessToken, isNull);
    expect(await tokens.readRefreshToken(), isNull);
  });

  test('keeps the access token in memory and only the refresh token in storage',
      () async {
    await tokens.save(access: 'access-1', refresh: 'refresh-1');

    expect(tokens.accessToken, 'access-1');
    expect(secure.values, {TokenStore.refreshTokenKey: 'refresh-1'});
  });

  test('reads the refresh token from storage every time, never from a copy',
      () async {
    await tokens.save(access: 'access-1', refresh: 'refresh-1');
    secure.values[TokenStore.refreshTokenKey] = 'written-elsewhere';

    expect(await tokens.readRefreshToken(), 'written-elsewhere');
  });

  test('clear leaves nothing behind', () async {
    await tokens.save(access: 'access-1', refresh: 'refresh-1');

    await tokens.clear();

    expect(tokens.accessToken, isNull);
    expect(secure.values, isEmpty);
  });

  test('lock drops the access token but keeps the refresh token stored',
      () async {
    await tokens.save(access: 'access-1', refresh: 'refresh-1');

    tokens.lock();

    expect(tokens.locked, isTrue);
    expect(tokens.accessToken, isNull);
    expect(secure.values[TokenStore.refreshTokenKey], 'refresh-1');
  });

  test('unlock opens the gate again', () {
    tokens.lock();
    tokens.unlock();

    expect(tokens.locked, isFalse);
  });
}
```

`test/core/session/fresh_install_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:wa_app/core/session/app_prefs.dart';
import 'package:wa_app/core/session/fresh_install.dart';

import '../../support/fakes.dart';

Future<AppPrefs> _prefs(Map<String, Object> initial) async {
  SharedPreferences.setMockInitialValues(initial);
  return AppPrefs(await SharedPreferences.getInstance());
}

void main() {
  test('wipes a token inherited from a previous install', () async {
    final prefs = await _prefs({});
    final secure = InMemorySecureStore()..values['wa.refresh'] = 'from-old-install';

    await wipeOnFreshInstall(prefs, secure);

    expect(secure.values, isEmpty);
    expect(prefs.installed, isTrue);
  });

  test('leaves the storage alone on every later start', () async {
    final prefs = await _prefs({'wa.installed': true});
    final secure = InMemorySecureStore()..values['wa.refresh'] = 'current';

    await wipeOnFreshInstall(prefs, secure);

    expect(secure.values, {'wa.refresh': 'current'});
  });
}
```

- [ ] **Step 2: Rodar os testes e confirmar o vermelho**

Run: `flutter test test/core/session`
Expected: FAIL, porque os módulos não existem.

- [ ] **Step 3: Escrever a implementação mínima**

`lib/core/session/secure_store.dart`:

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

/// The system keystore: Keychain on iOS, Keystore on Android. An interface so
/// tests can watch which keys are read.
abstract interface class SecureStore {
  Future<String?> read(String key);
  Future<void> write(String key, String value);
  Future<void> delete(String key);
  Future<void> deleteAll();
}

class FlutterSecureStore implements SecureStore {
  const FlutterSecureStore([this._storage = const FlutterSecureStorage()]);

  final FlutterSecureStorage _storage;

  @override
  Future<String?> read(String key) => _storage.read(key: key);

  @override
  Future<void> write(String key, String value) =>
      _storage.write(key: key, value: value);

  @override
  Future<void> delete(String key) => _storage.delete(key: key);

  @override
  Future<void> deleteAll() => _storage.deleteAll();
}
```

`lib/core/session/app_prefs.dart`:

```dart
import 'package:shared_preferences/shared_preferences.dart';

/// Flags that are not secret. They live outside the keystore on purpose: the
/// install marker must die with the app, which the iOS Keychain does not.
class AppPrefs {
  AppPrefs(this._prefs);

  final SharedPreferences _prefs;

  static const _installed = 'wa.installed';
  static const _hasSession = 'wa.hasSession';
  static const _biometricEnabled = 'wa.biometricEnabled';
  static const _biometricOffered = 'wa.biometricOffered';

  bool get installed => _prefs.getBool(_installed) ?? false;
  Future<void> markInstalled() => _prefs.setBool(_installed, true);

  /// Whether a session was stored, known without reading the refresh token.
  bool get hasSession => _prefs.getBool(_hasSession) ?? false;
  Future<void> setHasSession(bool value) => _prefs.setBool(_hasSession, value);

  bool get biometricEnabled => _prefs.getBool(_biometricEnabled) ?? false;
  Future<void> setBiometricEnabled(bool value) =>
      _prefs.setBool(_biometricEnabled, value);

  bool get biometricOffered => _prefs.getBool(_biometricOffered) ?? false;
  Future<void> markBiometricOffered() => _prefs.setBool(_biometricOffered, true);
}
```

`lib/core/session/fresh_install.dart`:

```dart
import 'package:wa_app/core/session/app_prefs.dart';
import 'package:wa_app/core/session/secure_store.dart';

/// The iOS Keychain survives uninstalling the app, so a reinstall would
/// rehydrate the previous install's session. The first start wipes it.
Future<void> wipeOnFreshInstall(AppPrefs prefs, SecureStore secure) async {
  if (prefs.installed) return;
  await secure.deleteAll();
  await prefs.markInstalled();
}
```

`lib/core/session/token_store.dart`:

```dart
import 'package:wa_app/core/session/secure_store.dart';

/// Where each half of the session lives.
///
/// The access token is short-lived and stays in memory. The refresh token is
/// the strong credential and goes to the system keystore. It is read from
/// storage every time, never cached, so whoever reads it always sees the latest
/// rotation.
///
/// [locked] is the biometric gate: while it is closed the access token is gone
/// and nothing may read the refresh token.
class TokenStore {
  TokenStore(this._secure);

  static const refreshTokenKey = 'wa.refresh';

  final SecureStore _secure;

  String? accessToken;
  bool locked = false;

  Future<String?> readRefreshToken() => _secure.read(refreshTokenKey);

  Future<void> save({required String access, required String refresh}) async {
    await _secure.write(refreshTokenKey, refresh);
    accessToken = access;
  }

  Future<void> clear() async {
    accessToken = null;
    await _secure.delete(refreshTokenKey);
  }

  void lock() {
    accessToken = null;
    locked = true;
  }

  void unlock() => locked = false;
}
```

- [ ] **Step 4: Rodar os testes e confirmar o verde**

Run: `flutter test test/core/session`
Expected: PASS, 8 testes.

- [ ] **Step 5: Commit**

```bash
git add lib/core/session test/core/session test/support/fakes.dart
git commit -m "feat(session): keep the access token in memory and the refresh token in the keystore" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 5: Renovação serializada

O maior risco da fatia. Os testes cobrem os riscos 1 e 2 da spec e o lado do refresh do risco 4.

**Files:**

- Create: `lib/core/session/token_refresher.dart`, `lib/core/session/session_expiry.dart`
- Create: `test/support/fake_adapter.dart`
- Test: `test/core/session/token_refresher_test.dart`

**Interfaces:**

- Consumes: `TokenStore`, `SerialLock`, `ApiError`, `AuthClient`.
- Produces:
  - `SessionExpiry` com `Stream<void> stream` e `void notify()`, entregues de forma síncrona.
  - `TokenRefresher({required TokenStore tokens, required AuthClient auth, required SessionExpiry expiry})` com `Future<String> refresh(String? failedAccessToken)`.
  - Em `test/support/fake_adapter.dart`: `FakeAdapter` com `on(method, path, handler)`, `requests`, `count(method, path)`; e os auxiliares `jsonResponse(body, {status})`, `emptyResponse({status})`, `unreachable(request)`.

- [ ] **Step 1: Escrever o adaptador falso**

`test/support/fake_adapter.dart`:

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:typed_data';

import 'package:dio/dio.dart';

typedef FakeHandler = FutureOr<ResponseBody> Function(RequestOptions request);

/// Answers HTTP calls from a route table keyed by method and path, and records
/// every request so tests can count them.
class FakeAdapter implements HttpClientAdapter {
  final Map<String, FakeHandler> _routes = {};
  final List<RequestOptions> requests = [];

  void on(String method, String path, FakeHandler handler) =>
      _routes['$method $path'] = handler;

  int count(String method, String path) =>
      requests.where((r) => r.method == method && r.path == path).length;

  @override
  Future<ResponseBody> fetch(
    RequestOptions options,
    Stream<Uint8List>? requestStream,
    Future<void>? cancelFuture,
  ) async {
    requests.add(options);
    final handler = _routes['${options.method} ${options.path}'];
    if (handler == null) {
      throw StateError('No fake route for ${options.method} ${options.path}');
    }
    return handler(options);
  }

  @override
  void close({bool force = false}) {}
}

ResponseBody jsonResponse(Object? body, {int status = 200}) =>
    ResponseBody.fromString(jsonEncode(body), status, headers: {
      Headers.contentTypeHeader: [Headers.jsonContentType],
    });

ResponseBody emptyResponse({int status = 204}) =>
    ResponseBody.fromString('', status);

Never unreachable(RequestOptions request) => throw DioException.connectionError(
      requestOptions: request,
      reason: 'connection refused',
    );
```

- [ ] **Step 2: Escrever o teste que falha**

`test/core/session/token_refresher_test.dart`:

```dart
import 'dart:async';

import 'package:dio/dio.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/http/api_error.dart';
import 'package:wa_app/core/session/session_expiry.dart';
import 'package:wa_app/core/session/token_refresher.dart';
import 'package:wa_app/core/session/token_store.dart';

import '../../support/fake_adapter.dart';
import '../../support/fakes.dart';

const _refreshPath = '/v1/sessions/refresh';

void main() {
  late FakeAdapter adapter;
  late InMemorySecureStore secure;
  late TokenStore tokens;
  late SessionExpiry expiry;
  late int expiredSignals;
  late TokenRefresher refresher;

  setUp(() {
    adapter = FakeAdapter();
    secure = InMemorySecureStore();
    tokens = TokenStore(secure);
    expiry = SessionExpiry();
    expiredSignals = 0;
    expiry.stream.listen((_) => expiredSignals++);
    final dio = Dio(BaseOptions(baseUrl: 'http://api.test'))
      ..httpClientAdapter = adapter;
    refresher =
        TokenRefresher(tokens: tokens, auth: AuthClient(dio), expiry: expiry);
  });

  Map<String, Object?> sentBody(int index) =>
      adapter.requests[index].data as Map<String, Object?>;

  test('rotates the pair and returns the new access token', () async {
    await tokens.save(access: 'expired', refresh: 'refresh-1');
    adapter.on('POST', _refreshPath,
        (_) => jsonResponse({'accessToken': 'fresh', 'refreshToken': 'refresh-2'}));

    final token = await refresher.refresh('expired');

    expect(token, 'fresh');
    expect(tokens.accessToken, 'fresh');
    expect(await tokens.readRefreshToken(), 'refresh-2');
    expect(sentBody(0), {'refreshToken': 'refresh-1'});
  });

  test('concurrent callers share one rotation', () async {
    await tokens.save(access: 'expired', refresh: 'refresh-1');
    final release = Completer<void>();
    adapter.on('POST', _refreshPath, (_) async {
      await release.future;
      return jsonResponse({'accessToken': 'fresh', 'refreshToken': 'refresh-2'});
    });

    final calls = List.generate(4, (_) => refresher.refresh('expired'));
    await Future<void>.delayed(const Duration(milliseconds: 10));
    release.complete();

    expect(await Future.wait(calls), everyElement('fresh'));
    expect(adapter.count('POST', _refreshPath), 1);
  });

  test('reads the refresh token inside the lock, so a queued rotation never '
      'replays a spent token', () async {
    await tokens.save(access: 'expired', refresh: 'refresh-1');
    var rotation = 0;
    adapter.on('POST', _refreshPath, (_) {
      rotation++;
      return jsonResponse(
          {'accessToken': 'fresh-$rotation', 'refreshToken': 'refresh-${rotation + 1}'});
    });

    // The second caller failed with the token the first one is about to issue,
    // so it really has to rotate again. It must send refresh-2, not refresh-1.
    final first = refresher.refresh('expired');
    final second = refresher.refresh('fresh-1');

    expect(await first, 'fresh-1');
    expect(await second, 'fresh-2');
    expect(sentBody(0), {'refreshToken': 'refresh-1'});
    expect(sentBody(1), {'refreshToken': 'refresh-2'});
  });

  test('a 401 from the refresh ends the session and says so once', () async {
    await tokens.save(access: 'expired', refresh: 'revoked');
    adapter.on('POST', _refreshPath,
        (_) => jsonResponse({'message': 'Invalid refresh token.'}, status: 401));

    await expectLater(
        refresher.refresh('expired'), throwsA(isA<SessionExpiredError>()));

    expect(await tokens.readRefreshToken(), isNull);
    expect(tokens.accessToken, isNull);
    expect(expiredSignals, 1);
  });

  test('a 403 from the refresh also ends the session', () async {
    await tokens.save(access: 'expired', refresh: 'refresh-1');
    adapter.on('POST', _refreshPath, (_) => jsonResponse({'message': 'x'}, status: 403));

    await expectLater(
        refresher.refresh('expired'), throwsA(isA<SessionExpiredError>()));
    expect(await tokens.readRefreshToken(), isNull);
  });

  test('a 5xx keeps the refresh token for a later attempt', () async {
    await tokens.save(access: 'expired', refresh: 'refresh-1');
    adapter.on('POST', _refreshPath, (_) => jsonResponse(null, status: 502));

    await expectLater(refresher.refresh('expired'),
        throwsA(isA<ApiError>().having((e) => e.status, 'status', 502)));

    expect(await tokens.readRefreshToken(), 'refresh-1');
    expect(expiredSignals, 0);
  });

  test('an unreachable server keeps the refresh token too', () async {
    await tokens.save(access: 'expired', refresh: 'refresh-1');
    adapter.on('POST', _refreshPath, unreachable);

    await expectLater(refresher.refresh('expired'),
        throwsA(isA<ApiError>().having((e) => e.status, 'status', 0)));
    expect(await tokens.readRefreshToken(), 'refresh-1');
  });

  test('with no stored session it fails without calling the API or signalling',
      () async {
    await expectLater(refresher.refresh(null), throwsA(isA<SessionExpiredError>()));

    expect(adapter.requests, isEmpty);
    expect(expiredSignals, 0);
  });

  test('while locked it refuses to run and never reads the refresh token',
      () async {
    await tokens.save(access: 'access-1', refresh: 'refresh-1');
    tokens.lock();
    secure.reads.clear();

    await expectLater(refresher.refresh(null), throwsA(isA<SessionLockedError>()));

    expect(secure.reads, isNot(contains(TokenStore.refreshTokenKey)));
    expect(adapter.requests, isEmpty);
  });
}
```

- [ ] **Step 3: Rodar o teste e confirmar o vermelho**

Run: `flutter test test/core/session/token_refresher_test.dart`
Expected: FAIL, porque `token_refresher.dart` e `session_expiry.dart` não existem.

- [ ] **Step 4: Escrever a implementação mínima**

`lib/core/session/session_expiry.dart`:

```dart
import 'dart:async';

/// Signals that the server ended the session. Delivered synchronously, so
/// whoever listens has already reacted when the refresh call throws.
class SessionExpiry {
  final StreamController<void> _controller =
      StreamController<void>.broadcast(sync: true);

  Stream<void> get stream => _controller.stream;

  void notify() => _controller.add(null);
}
```

`lib/core/session/token_refresher.dart`:

```dart
import 'package:dio/dio.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/http/api_error.dart';
import 'package:wa_app/core/http/serial_lock.dart';
import 'package:wa_app/core/session/session_expiry.dart';
import 'package:wa_app/core/session/token_store.dart';

/// Rotates the token pair, one rotation at a time.
///
/// The API reads a refresh token that comes back after being rotated as theft
/// and revokes every session the user has, on every device. Two rotations
/// crossing would log the operator out of the app and the web at once. Two
/// rules keep that from happening:
///
/// - The refresh token is read from storage inside the lock, never captured
///   before waiting for it, so a queued caller sends the latest token.
/// - A caller that finds an access token different from the one that failed
///   knows someone already rotated, and reuses it instead of rotating again.
class TokenRefresher {
  TokenRefresher({
    required TokenStore tokens,
    required AuthClient auth,
    required SessionExpiry expiry,
  })  : _tokens = tokens,
        _auth = auth,
        _expiry = expiry;

  final TokenStore _tokens;
  final AuthClient _auth;
  final SessionExpiry _expiry;
  final SerialLock _lock = SerialLock();

  Future<String> refresh(String? failedAccessToken) => _lock.run(() async {
        if (_tokens.locked) throw const SessionLockedError();

        final current = _tokens.accessToken;
        if (current != null && current != failedAccessToken) return current;

        final refreshToken = await _tokens.readRefreshToken();
        if (refreshToken == null) throw const SessionExpiredError();

        try {
          final pair = await _auth.postV1SessionsRefresh(
            body: RefreshBody(refreshToken: refreshToken),
          );
          await _tokens.save(access: pair.accessToken, refresh: pair.refreshToken);
          return pair.accessToken;
        } on DioException catch (exception) {
          final status = exception.response?.statusCode;
          // Only the API refusing the token means "this is no longer yours".
          // A restart or a dropped connection keeps it for the next attempt.
          if (status == 401 || status == 403) {
            await _tokens.clear();
            _expiry.notify();
            throw const SessionExpiredError();
          }
          throw ApiError.fromDio(exception);
        }
      });
}
```

- [ ] **Step 5: Rodar o teste e confirmar o verde**

Run: `flutter test test/core/session/token_refresher_test.dart`
Expected: PASS, 9 testes.

- [ ] **Step 6: Commit**

```bash
git add lib/core/session test/core/session test/support/fake_adapter.dart
git commit -m "feat(session): rotate the token pair one caller at a time" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 6: Interceptador de autenticação e providers HTTP

**Files:**

- Create: `lib/core/http/auth_interceptor.dart`, `lib/core/http/http_providers.dart`, `lib/core/http/retry_policy.dart`
- Test: `test/core/http/auth_interceptor_test.dart`, `test/core/http/retry_policy_test.dart`

**Interfaces:**

- Consumes: `TokenRefresher.refresh` (por função, sem importar `core/session`), `ApiError`.
- Produces:
  - `AuthInterceptor({required String? Function() accessToken, required Future<String> Function(String? failed) refresh, required Dio retry})`.
  - `envProvider`, `httpClientAdapterProvider` (substituível em teste), `bareDioProvider`, `Dio newDio(Ref ref)`.
  - `Duration? retryPolicy(int retryCount, Object error)`.

- [ ] **Step 1: Escrever os testes que falham**

`test/core/http/auth_interceptor_test.dart`:

```dart
import 'dart:async';

import 'package:dio/dio.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/http/api_error.dart';
import 'package:wa_app/core/http/auth_interceptor.dart';

import '../../support/fake_adapter.dart';

void main() {
  late FakeAdapter adapter;
  late Dio dio;
  String? accessToken;
  late int refreshCalls;
  late Future<String> Function(String? failed) refresh;

  setUp(() {
    adapter = FakeAdapter();
    accessToken = 'expired';
    refreshCalls = 0;
    refresh = (failed) async {
      refreshCalls++;
      accessToken = 'fresh';
      return 'fresh';
    };
    dio = Dio(BaseOptions(baseUrl: 'http://api.test'))..httpClientAdapter = adapter;
    dio.interceptors.add(AuthInterceptor(
      accessToken: () => accessToken,
      refresh: (failed) => refresh(failed),
      retry: dio,
    ));
  });

  FakeHandler acceptsOnly(String token) => (request) =>
      request.headers['Authorization'] == 'Bearer $token'
          ? jsonResponse([])
          : jsonResponse({'message': 'Authentication required.'}, status: 401);

  test('sends the access token as a Bearer header', () async {
    accessToken = 'valid';
    adapter.on('GET', '/v1/suppliers', acceptsOnly('valid'));

    final response = await dio.get<Object?>('/v1/suppliers');

    expect(response.statusCode, 200);
    expect(refreshCalls, 0);
  });

  test('a 401 refreshes once and retries the request with the new token',
      () async {
    adapter.on('GET', '/v1/suppliers', acceptsOnly('fresh'));

    final response = await dio.get<Object?>('/v1/suppliers');

    expect(response.statusCode, 200);
    expect(refreshCalls, 1);
    expect(adapter.count('GET', '/v1/suppliers'), 2);
  });

  test('passes the token that failed to the refresher', () async {
    String? seen;
    refresh = (failed) async {
      seen = failed;
      accessToken = 'fresh';
      return 'fresh';
    };
    adapter.on('GET', '/v1/suppliers', acceptsOnly('fresh'));

    await dio.get<Object?>('/v1/suppliers');

    expect(seen, 'expired');
  });

  test('four parallel 401s are all retried and answered', () async {
    final release = Completer<void>();
    refresh = (failed) async {
      refreshCalls++;
      await release.future;
      accessToken = 'fresh';
      return 'fresh';
    };
    adapter.on('GET', '/v1/suppliers', acceptsOnly('fresh'));

    final calls = List.generate(4, (_) => dio.get<Object?>('/v1/suppliers'));
    await Future<void>.delayed(const Duration(milliseconds: 10));
    release.complete();
    final responses = await Future.wait(calls);

    expect(responses.map((r) => r.statusCode), everyElement(200));
  });

  test('a second 401 after a refresh is the endpoint answer and passes through',
      () async {
    adapter.on('PATCH', '/v1/sessions/me/password',
        (_) => jsonResponse({'message': 'Invalid credentials.'}, status: 401));

    await expectLater(
      dio.patch<Object?>('/v1/sessions/me/password'),
      throwsA(isA<DioException>()
          .having((e) => e.response?.statusCode, 'status', 401)),
    );
    expect(refreshCalls, 1);
    expect(adapter.count('PATCH', '/v1/sessions/me/password'), 2);
  });

  test('a failed refresh rejects with the ApiError it raised', () async {
    refresh = (_) async => throw const SessionExpiredError();
    adapter.on('GET', '/v1/suppliers', acceptsOnly('fresh'));

    await expectLater(
      dio.get<Object?>('/v1/suppliers'),
      throwsA(isA<DioException>()
          .having((e) => e.error, 'error', isA<SessionExpiredError>())),
    );
  });

  test('statuses other than 401 do not touch the session', () async {
    accessToken = 'valid';
    adapter.on('GET', '/v1/suppliers', (_) => jsonResponse({'message': 'x'}, status: 403));

    await expectLater(dio.get<Object?>('/v1/suppliers'), throwsA(isA<DioException>()));
    expect(refreshCalls, 0);
  });
}
```

`test/core/http/retry_policy_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/http/api_error.dart';
import 'package:wa_app/core/http/retry_policy.dart';

void main() {
  test('retries an unreachable server with growing waits, at most twice', () {
    expect(retryPolicy(0, const ApiError(0)), const Duration(milliseconds: 500));
    expect(retryPolicy(1, const ApiError(0)), const Duration(milliseconds: 1000));
    expect(retryPolicy(2, const ApiError(0)), isNull);
  });

  test('retries a 5xx', () {
    expect(retryPolicy(0, const ApiError(503)), isNotNull);
  });

  test('never retries a 4xx: asking again will not change the answer', () {
    expect(retryPolicy(0, const ApiError(403)), isNull);
    expect(retryPolicy(0, const ApiError(404)), isNull);
  });

  test('never retries an ended or locked session', () {
    expect(retryPolicy(0, const SessionExpiredError()), isNull);
    expect(retryPolicy(0, const SessionLockedError()), isNull);
  });

  test('never retries an error that is not from the API', () {
    expect(retryPolicy(0, StateError('bug')), isNull);
  });
}
```

- [ ] **Step 2: Rodar os testes e confirmar o vermelho**

Run: `flutter test test/core/http`
Expected: FAIL, porque `auth_interceptor.dart` e `retry_policy.dart` não existem.

- [ ] **Step 3: Escrever a implementação mínima**

`lib/core/http/auth_interceptor.dart`:

```dart
import 'package:dio/dio.dart';
import 'package:wa_app/core/http/api_error.dart';

/// Adds the access token to every request and, on a 401, refreshes once and
/// retries once. It knows nothing about where tokens live: the session hands
/// it two functions.
class AuthInterceptor extends Interceptor {
  AuthInterceptor({
    required String? Function() accessToken,
    required Future<String> Function(String? failed) refresh,
    required Dio retry,
  })  : _accessToken = accessToken,
        _refresh = refresh,
        _retry = retry;

  static const _sentTokenKey = 'wa.sentAccessToken';
  static const _retriedKey = 'wa.retried';

  final String? Function() _accessToken;
  final Future<String> Function(String? failed) _refresh;
  final Dio _retry;

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final token = _accessToken();
    options.extra[_sentTokenKey] = token;
    if (token != null) options.headers['Authorization'] = 'Bearer $token';
    handler.next(options);
  }

  @override
  Future<void> onError(
    DioException err,
    ErrorInterceptorHandler handler,
  ) async {
    final request = err.requestOptions;
    // A second 401, on a token the API just issued, is the endpoint's own
    // answer, like PATCH /sessions/me/password with a wrong current password.
    if (err.response?.statusCode != 401 || request.extra[_retriedKey] == true) {
      return handler.next(err);
    }
    try {
      await _refresh(request.extra[_sentTokenKey] as String?);
      request.extra[_retriedKey] = true;
      handler.resolve(await _retry.fetch<Object?>(request));
    } on ApiError catch (error) {
      handler.reject(DioException(
        requestOptions: request,
        response: err.response,
        error: error,
      ));
    } on DioException catch (retryError) {
      handler.reject(retryError);
    }
  }
}
```

`lib/core/http/http_providers.dart`:

```dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:wa_app/core/env/env.dart';

final envProvider = Provider<Env>(
  (ref) => throw UnimplementedError('envProvider is overridden in main'),
);

/// Overridden in tests with a fake; null means the real network.
final httpClientAdapterProvider = Provider<HttpClientAdapter?>((ref) => null);

Dio newDio(Ref ref) {
  final dio = Dio(BaseOptions(
    baseUrl: ref.watch(envProvider).apiOrigin.toString(),
    connectTimeout: const Duration(seconds: 10),
    receiveTimeout: const Duration(seconds: 15),
    sendTimeout: const Duration(seconds: 15),
  ));
  final adapter = ref.watch(httpClientAdapterProvider);
  if (adapter != null) dio.httpClientAdapter = adapter;
  return dio;
}

/// No session attached: sign-in and refresh go through here, so the refresh
/// can never trigger itself.
final bareDioProvider = Provider<Dio>(newDio);
```

`lib/core/http/retry_policy.dart`:

```dart
import 'package:wa_app/core/http/api_error.dart';

/// When Riverpod retries a provider that failed. Only an unreachable server or
/// a 5xx is worth asking again; a 4xx will answer the same.
Duration? retryPolicy(int retryCount, Object error) {
  if (retryCount >= 2) return null;
  if (error is! ApiError) return null;
  if (error is SessionExpiredError || error is SessionLockedError) return null;
  final transient = error.status == 0 || error.status >= 500;
  return transient ? Duration(milliseconds: 500 * (1 << retryCount)) : null;
}
```

- [ ] **Step 4: Rodar os testes e confirmar o verde**

Run: `flutter test test/core/http`
Expected: PASS, 21 testes (os 9 da Task 3, 7 do interceptador e 5 da política).

- [ ] **Step 5: Commit**

```bash
git add lib/core/http test/core/http
git commit -m "feat(http): refresh once and retry once when a request gets a 401" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 7: Sessão: estados, controlador e providers

**Files:**

- Create: `lib/core/session/biometric_auth.dart`, `lib/core/session/session_state.dart`, `lib/core/session/session_controller.dart`, `lib/core/session/session_providers.dart`, `lib/core/api/api_providers.dart`
- Modify: `test/support/fakes.dart` (acrescenta `FakeBiometricAuth`)
- Create: `test/support/harness.dart`
- Test: `test/core/session/session_controller_test.dart`

**Interfaces:**

- Consumes: tudo de `core/http` e `core/session` até aqui.
- Produces:
  - `BiometricAuth` (interface: `isAvailable()`, `authenticate()`) e `LocalBiometricAuth`.
  - Estados `SessionBooting`, `SessionUnreachable`, `SessionLocked`, `SessionSignedOut({String? notice})`, `SessionSignedIn(CurrentUser user)` com `bool can(Permission)`.
  - `sessionProvider` (`NotifierProvider<SessionController, SessionState>`), com os métodos `boot()`, `retry()`, `signIn(username, password)`, `signOut()`, `changePassword(current, next)`, `unlock()`, `usePassword()`, `onBackgrounded()`, `onForegrounded()`, `shouldOfferBiometric()`, `markBiometricOffered()`, `enableBiometric()`, `disableBiometric()`.
  - `sharedPreferencesProvider`, `secureStoreProvider`, `appPrefsProvider`, `tokenStoreProvider`, `sessionExpiryProvider`, `tokenRefresherProvider`, `biometricAuthProvider`, `biometricAvailableProvider`, `clockProvider`.
  - Constantes `biometricLockAfter` (5 min), `sessionExpiredNotice`, `passwordChangedNotice`.
  - `dioProvider` (autenticado) e `apiProvider`.
  - Em `test/support/harness.dart`: `makeContainer(...)`.

- [ ] **Step 1: Completar os falsos e o harness**

Acrescentar a `test/support/fakes.dart` (com `import 'dart:async';` no topo do arquivo):

```dart
import 'package:wa_app/core/session/biometric_auth.dart';

class FakeBiometricAuth implements BiometricAuth {
  bool available = true;
  bool succeeds = true;
  int prompts = 0;

  /// When set, the prompt stays open until the test completes it.
  Completer<bool>? pending;

  @override
  Future<bool> isAvailable() async => available;

  @override
  Future<bool> authenticate() async {
    prompts++;
    return pending?.future ?? succeeds;
  }
}
```

`test/support/harness.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:wa_app/core/env/env.dart';
import 'package:wa_app/core/http/http_providers.dart';
import 'package:wa_app/core/session/session_providers.dart';

import 'fake_adapter.dart';
import 'fakes.dart';

/// A container wired to fakes. Retries are off so a failing call fails at
/// once; the policy itself is tested on its own.
Future<ProviderContainer> makeContainer({
  required FakeAdapter adapter,
  InMemorySecureStore? secure,
  FakeBiometricAuth? biometric,
  Map<String, Object> prefs = const {'wa.installed': true},
  DateTime Function()? clock,
}) async {
  SharedPreferences.setMockInitialValues(prefs);
  final sharedPreferences = await SharedPreferences.getInstance();
  final container = ProviderContainer(
    retry: (_, _) => null,
    overrides: [
      envProvider.overrideWithValue(Env.parse('http://api.test')),
      httpClientAdapterProvider.overrideWithValue(adapter),
      sharedPreferencesProvider.overrideWithValue(sharedPreferences),
      secureStoreProvider.overrideWithValue(secure ?? InMemorySecureStore()),
      biometricAuthProvider.overrideWithValue(biometric ?? FakeBiometricAuth()),
      if (clock != null) clockProvider.overrideWithValue(clock),
    ],
  );
  addTearDown(container.dispose);
  return container;
}
```

- [ ] **Step 2: Escrever o teste que falha**

`test/core/session/session_controller_test.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/http/api_error.dart';
import 'package:wa_app/core/session/session_controller.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/core/session/session_state.dart';
import 'package:wa_app/core/session/token_store.dart';

import '../../support/fake_adapter.dart';
import '../../support/fakes.dart';
import '../../support/harness.dart';
import '../../support/samples.dart';

void main() {
  late FakeAdapter adapter;
  late InMemorySecureStore secure;
  late FakeBiometricAuth biometric;

  setUp(() {
    adapter = FakeAdapter()
      ..on('GET', '/v1/sessions/me',
          (_) => jsonResponse(aCurrentUser(permissions: [Permission.catalogRead]).toJson()))
      ..on('POST', '/v1/sessions/refresh',
          (_) => jsonResponse(aTokenPair(access: 'access-2', refresh: 'refresh-2').toJson()))
      ..on('POST', '/v1/sessions/signin', (_) => jsonResponse(aTokenPair().toJson()))
      ..on('POST', '/v1/sessions/signout', (_) => emptyResponse());
    secure = InMemorySecureStore();
    biometric = FakeBiometricAuth();
  });

  Future<ProviderContainer> container({
    Map<String, Object> prefs = const {'wa.installed': true},
    DateTime Function()? clock,
  }) =>
      makeContainer(
          adapter: adapter, secure: secure, biometric: biometric, prefs: prefs, clock: clock);

  SessionController controllerOf(ProviderContainer c) => c.read(sessionProvider.notifier);

  group('boot', () {
    test('with no stored session goes to sign-in without a notice', () async {
      final c = await container();

      await controllerOf(c).boot();

      expect(c.read(sessionProvider), isA<SessionSignedOut>()
          .having((s) => s.notice, 'notice', isNull));
      expect(adapter.requests, isEmpty);
    });

    test('rehydrates a stored session with one refresh', () async {
      secure.values[TokenStore.refreshTokenKey] = 'refresh-1';
      final c = await container(prefs: {'wa.installed': true, 'wa.hasSession': true});

      await controllerOf(c).boot();

      final state = c.read(sessionProvider);
      expect(state, isA<SessionSignedIn>());
      expect((state as SessionSignedIn).can(Permission.catalogRead), isTrue);
      expect(state.can(Permission.catalogCreate), isFalse);
      expect(adapter.count('POST', '/v1/sessions/refresh'), 1);
    });

    test('a revoked session goes to sign-in saying it expired', () async {
      secure.values[TokenStore.refreshTokenKey] = 'revoked';
      adapter.on('POST', '/v1/sessions/refresh',
          (_) => jsonResponse({'message': 'x'}, status: 401));
      final c = await container(prefs: {'wa.installed': true, 'wa.hasSession': true});

      await controllerOf(c).boot();

      expect(c.read(sessionProvider), isA<SessionSignedOut>()
          .having((s) => s.notice, 'notice', sessionExpiredNotice));
    });

    test('an unreachable server keeps the token and offers to retry', () async {
      secure.values[TokenStore.refreshTokenKey] = 'refresh-1';
      adapter.on('POST', '/v1/sessions/refresh', unreachable);
      final c = await container(prefs: {'wa.installed': true, 'wa.hasSession': true});

      await controllerOf(c).boot();

      expect(c.read(sessionProvider), isA<SessionUnreachable>());
      expect(secure.values[TokenStore.refreshTokenKey], 'refresh-1');

      adapter.on('POST', '/v1/sessions/refresh',
          (_) => jsonResponse(aTokenPair(access: 'a', refresh: 'r').toJson()));
      await controllerOf(c).retry();
      expect(c.read(sessionProvider), isA<SessionSignedIn>());
    });

    test('wipes a token inherited from a previous install', () async {
      secure.values[TokenStore.refreshTokenKey] = 'from-old-install';
      final c = await container(prefs: {});

      await controllerOf(c).boot();

      expect(secure.values, isEmpty);
      expect(c.read(sessionProvider), isA<SessionSignedOut>());
    });
  });

  group('biometric gate', () {
    const lockedPrefs = {
      'wa.installed': true,
      'wa.hasSession': true,
      'wa.biometricEnabled': true,
    };

    test('boot locks without reading the refresh token', () async {
      secure.values[TokenStore.refreshTokenKey] = 'refresh-1';
      final c = await container(prefs: lockedPrefs);

      await controllerOf(c).boot();

      expect(c.read(sessionProvider), isA<SessionLocked>());
      expect(secure.reads, isNot(contains(TokenStore.refreshTokenKey)));
      expect(adapter.requests, isEmpty);
    });

    test('a failed biometric goes to password sign-in, still without reading it',
        () async {
      secure.values[TokenStore.refreshTokenKey] = 'refresh-1';
      biometric.succeeds = false;
      final c = await container(prefs: lockedPrefs);
      await controllerOf(c).boot();

      await controllerOf(c).unlock();

      expect(c.read(sessionProvider), isA<SessionSignedOut>());
      expect(secure.reads, isNot(contains(TokenStore.refreshTokenKey)));
    });

    test('a valid biometric rehydrates the session', () async {
      secure.values[TokenStore.refreshTokenKey] = 'refresh-1';
      final c = await container(prefs: lockedPrefs);
      await controllerOf(c).boot();

      await controllerOf(c).unlock();

      expect(c.read(sessionProvider), isA<SessionSignedIn>());
    });

    test('a device that lost its biometrics turns the gate off and asks for the '
        'password', () async {
      secure.values[TokenStore.refreshTokenKey] = 'refresh-1';
      biometric.available = false;
      final c = await container(prefs: lockedPrefs);
      await controllerOf(c).boot();

      await controllerOf(c).unlock();

      expect(c.read(sessionProvider), isA<SessionSignedOut>());
      expect(c.read(appPrefsProvider).biometricEnabled, isFalse);
      expect(secure.reads, isNot(contains(TokenStore.refreshTokenKey)));
    });

    test('five minutes in the background locks and drops the access token',
        () async {
      var now = DateTime(2026, 10, 19, 8);
      secure.values[TokenStore.refreshTokenKey] = 'refresh-1';
      final c = await container(
          prefs: {'wa.installed': true, 'wa.hasSession': true}, clock: () => now);
      await controllerOf(c).boot();
      await controllerOf(c).enableBiometric();

      controllerOf(c).onBackgrounded();
      now = now.add(biometricLockAfter);
      controllerOf(c).onForegrounded();

      expect(c.read(sessionProvider), isA<SessionLocked>());
      expect(c.read(tokenStoreProvider).accessToken, isNull);
    });

    test('a short trip to the background does not lock', () async {
      var now = DateTime(2026, 10, 19, 8);
      secure.values[TokenStore.refreshTokenKey] = 'refresh-1';
      final c = await container(
          prefs: {'wa.installed': true, 'wa.hasSession': true}, clock: () => now);
      await controllerOf(c).boot();
      await controllerOf(c).enableBiometric();

      controllerOf(c).onBackgrounded();
      now = now.add(const Duration(minutes: 4));
      controllerOf(c).onForegrounded();

      expect(c.read(sessionProvider), isA<SessionSignedIn>());
    });

    test('without the gate turned on, the background never locks', () async {
      var now = DateTime(2026, 10, 19, 8);
      secure.values[TokenStore.refreshTokenKey] = 'refresh-1';
      final c = await container(
          prefs: {'wa.installed': true, 'wa.hasSession': true}, clock: () => now);
      await controllerOf(c).boot();

      controllerOf(c).onBackgrounded();
      now = now.add(const Duration(hours: 1));
      controllerOf(c).onForegrounded();

      expect(c.read(sessionProvider), isA<SessionSignedIn>());
    });

    test('offers the gate once, only on a device that supports it', () async {
      final c = await container();

      expect(await controllerOf(c).shouldOfferBiometric(), isTrue);
      await controllerOf(c).markBiometricOffered();
      expect(await controllerOf(c).shouldOfferBiometric(), isFalse);
    });

    test('does not offer the gate on a device without biometrics', () async {
      biometric.available = false;
      final c = await container();

      expect(await controllerOf(c).shouldOfferBiometric(), isFalse);
    });
  });

  group('sign in and out', () {
    test('signing in stores the pair and loads the user', () async {
      final c = await container();
      await controllerOf(c).boot();

      await controllerOf(c).signIn('ana', 'secret');

      expect(c.read(sessionProvider), isA<SessionSignedIn>());
      expect(secure.values[TokenStore.refreshTokenKey], 'refresh-1');
      expect(c.read(appPrefsProvider).hasSession, isTrue);
      expect(adapter.requests.first.data, {'username': 'ana', 'password': 'secret'});
    });

    test('wrong credentials surface as a 401 and leave the state alone', () async {
      adapter.on('POST', '/v1/sessions/signin',
          (_) => jsonResponse({'message': 'Invalid credentials.'}, status: 401));
      final c = await container();
      await controllerOf(c).boot();

      await expectLater(controllerOf(c).signIn('ana', 'wrong'),
          throwsA(isA<ApiError>().having((e) => e.status, 'status', 401)));
      expect(c.read(sessionProvider), isA<SessionSignedOut>());
    });

    test('signing out revokes the refresh token and clears everything', () async {
      final c = await container();
      await controllerOf(c).boot();
      await controllerOf(c).signIn('ana', 'secret');

      await controllerOf(c).signOut();

      expect(adapter.count('POST', '/v1/sessions/signout'), 1);
      expect(secure.values, isEmpty);
      expect(c.read(sessionProvider), isA<SessionSignedOut>());
      expect(c.read(appPrefsProvider).hasSession, isFalse);
    });

    test('signing out clears locally even when the API is unreachable', () async {
      adapter.on('POST', '/v1/sessions/signout', unreachable);
      final c = await container();
      await controllerOf(c).boot();
      await controllerOf(c).signIn('ana', 'secret');

      await controllerOf(c).signOut();

      expect(secure.values, isEmpty);
      expect(c.read(sessionProvider), isA<SessionSignedOut>());
    });

    test('changing the password ends the session with its own notice', () async {
      adapter.on('PATCH', '/v1/sessions/me/password', (_) => emptyResponse());
      final c = await container();
      await controllerOf(c).boot();
      await controllerOf(c).signIn('ana', 'secret');

      await controllerOf(c).changePassword('secret', 'new-secret');

      expect(secure.values, isEmpty);
      expect(c.read(sessionProvider), isA<SessionSignedOut>()
          .having((s) => s.notice, 'notice', passwordChangedNotice));
    });
  });
}
```

- [ ] **Step 3: Rodar o teste e confirmar o vermelho**

Run: `flutter test test/core/session/session_controller_test.dart`
Expected: FAIL, porque os módulos da sessão não existem.

- [ ] **Step 4: Escrever a implementação mínima**

`lib/core/session/biometric_auth.dart`:

```dart
import 'package:local_auth/local_auth.dart';

/// The device's fingerprint or face check. A local gate over the stored
/// session: the API never knows it exists.
abstract interface class BiometricAuth {
  Future<bool> isAvailable();
  Future<bool> authenticate();
}

class LocalBiometricAuth implements BiometricAuth {
  LocalBiometricAuth([LocalAuthentication? auth])
      : _auth = auth ?? LocalAuthentication();

  final LocalAuthentication _auth;

  @override
  Future<bool> isAvailable() async {
    try {
      return await _auth.isDeviceSupported() &&
          (await _auth.getAvailableBiometrics()).isNotEmpty;
    } on Exception {
      return false;
    }
  }

  @override
  Future<bool> authenticate() async {
    try {
      return await _auth.authenticate(
        localizedReason: 'Confirme que é você para abrir o wa-system.',
        biometricOnly: true,
      );
    } on Exception {
      return false;
    }
  }
}
```

`lib/core/session/session_state.dart`:

```dart
import 'package:wa_app/core/api/api.dart';

sealed class SessionState {
  const SessionState();
}

/// Rehydrating at startup.
final class SessionBooting extends SessionState {
  const SessionBooting();
}

/// The server could not be reached at startup; the stored session is kept.
final class SessionUnreachable extends SessionState {
  const SessionUnreachable();
}

/// The biometric gate is closed.
final class SessionLocked extends SessionState {
  const SessionLocked();
}

final class SessionSignedOut extends SessionState {
  const SessionSignedOut({this.notice});

  /// Why the user is here, when it is not obvious: an expired session or a
  /// password just changed.
  final String? notice;
}

final class SessionSignedIn extends SessionState {
  const SessionSignedIn(this.user);

  final CurrentUser user;

  /// Mirrors the API's decision so the UI does not offer what would answer
  /// 403. The API stays the authority.
  bool can(Permission permission) => user.permissions.contains(permission);
}
```

`lib/core/session/session_providers.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/http/http_providers.dart';
import 'package:wa_app/core/session/app_prefs.dart';
import 'package:wa_app/core/session/biometric_auth.dart';
import 'package:wa_app/core/session/secure_store.dart';
import 'package:wa_app/core/session/session_controller.dart';
import 'package:wa_app/core/session/session_expiry.dart';
import 'package:wa_app/core/session/session_state.dart';
import 'package:wa_app/core/session/token_refresher.dart';
import 'package:wa_app/core/session/token_store.dart';

final sharedPreferencesProvider = Provider<SharedPreferences>(
  (ref) => throw UnimplementedError('sharedPreferencesProvider is overridden in main'),
);

final secureStoreProvider = Provider<SecureStore>((ref) => const FlutterSecureStore());

final appPrefsProvider =
    Provider<AppPrefs>((ref) => AppPrefs(ref.watch(sharedPreferencesProvider)));

final tokenStoreProvider =
    Provider<TokenStore>((ref) => TokenStore(ref.watch(secureStoreProvider)));

final sessionExpiryProvider = Provider<SessionExpiry>((ref) => SessionExpiry());

final tokenRefresherProvider = Provider<TokenRefresher>((ref) => TokenRefresher(
      tokens: ref.watch(tokenStoreProvider),
      auth: AuthClient(ref.watch(bareDioProvider)),
      expiry: ref.watch(sessionExpiryProvider),
    ));

final biometricAuthProvider = Provider<BiometricAuth>((ref) => LocalBiometricAuth());

final biometricAvailableProvider = FutureProvider<bool>(
  (ref) => ref.watch(biometricAuthProvider).isAvailable(),
);

final clockProvider = Provider<DateTime Function()>((ref) => DateTime.now);

final sessionProvider =
    NotifierProvider<SessionController, SessionState>(SessionController.new);
```

`lib/core/api/api_providers.dart`:

```dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/http/auth_interceptor.dart';
import 'package:wa_app/core/http/http_providers.dart';
import 'package:wa_app/core/session/session_providers.dart';

/// Every call made on behalf of a signed-in user goes through this client.
final dioProvider = Provider<Dio>((ref) {
  final dio = newDio(ref);
  final tokens = ref.watch(tokenStoreProvider);
  final refresher = ref.watch(tokenRefresherProvider);
  dio.interceptors.add(AuthInterceptor(
    accessToken: () => tokens.accessToken,
    refresh: refresher.refresh,
    retry: dio,
  ));
  return dio;
});

final apiProvider = Provider<WaApi>((ref) => WaApi(ref.watch(dioProvider)));
```

`lib/core/session/session_controller.dart`:

```dart
import 'dart:async';

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/api/api_providers.dart';
import 'package:wa_app/core/http/api_error.dart';
import 'package:wa_app/core/session/fresh_install.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/core/session/session_state.dart';

/// How long the app may stay in the background before the biometric gate
/// closes again. One constant until real use asks for another number.
const biometricLockAfter = Duration(minutes: 5);

const sessionExpiredNotice = 'Sua sessão expirou. Entre novamente.';
const passwordChangedNotice = 'Senha trocada. Entre de novo com a senha nova.';

class SessionController extends Notifier<SessionState> {
  DateTime? _backgroundedAt;

  @override
  SessionState build() {
    final subscription = ref.read(sessionExpiryProvider).stream.listen((_) {
      unawaited(ref.read(appPrefsProvider).setHasSession(false));
      state = const SessionSignedOut(notice: sessionExpiredNotice);
    });
    ref.onDispose(subscription.cancel);
    return const SessionBooting();
  }

  Future<void> boot() async {
    final prefs = ref.read(appPrefsProvider);
    await wipeOnFreshInstall(prefs, ref.read(secureStoreProvider));
    if (!prefs.hasSession) {
      state = const SessionSignedOut();
      return;
    }
    if (prefs.biometricEnabled) {
      ref.read(tokenStoreProvider).lock();
      state = const SessionLocked();
      return;
    }
    await _rehydrate();
  }

  Future<void> retry() => _rehydrate();

  Future<void> signIn(String username, String password) async {
    final pair = await callApi(() => AuthClient(ref.read(bareDioProvider))
        .postV1SessionsSignin(body: SignInBody(username: username, password: password)));
    final tokens = ref.read(tokenStoreProvider);
    await tokens.save(access: pair.accessToken, refresh: pair.refreshToken);
    tokens.unlock();
    await ref.read(appPrefsProvider).setHasSession(true);
    state = SessionSignedIn(await _fetchUser());
  }

  Future<void> signOut() async {
    final refreshToken = await ref.read(tokenStoreProvider).readRefreshToken();
    if (refreshToken != null) {
      try {
        await callApi(() => ref
            .read(apiProvider)
            .auth
            .postV1SessionsSignout(body: SignOutBody(refreshToken: refreshToken)));
      } on ApiError {
        // Signing out has to sign out, whatever the API answers.
      }
    }
    await _endLocally();
  }

  /// The API revokes every refresh token on a password change, so the session
  /// ends here too. A wrong current password surfaces as a 401 ApiError.
  Future<void> changePassword(String current, String next) async {
    await callApi(() => ref.read(apiProvider).auth.patchV1SessionsMePassword(
        body: ChangePasswordBody(currentPassword: current, newPassword: next)));
    await _endLocally(notice: passwordChangedNotice);
  }

  Future<void> unlock() async {
    final biometric = ref.read(biometricAuthProvider);
    if (!await biometric.isAvailable()) {
      await ref.read(appPrefsProvider).setBiometricEnabled(false);
      state = const SessionSignedOut();
      return;
    }
    if (!await biometric.authenticate()) {
      state = const SessionSignedOut();
      return;
    }
    ref.read(tokenStoreProvider).unlock();
    await _rehydrate();
  }

  /// Leaves the lock screen for the password; the stored session is not
  /// touched, and signing in replaces it.
  void usePassword() => state = const SessionSignedOut();

  void onBackgrounded() => _backgroundedAt = ref.read(clockProvider)();

  void onForegrounded() {
    final since = _backgroundedAt;
    _backgroundedAt = null;
    if (since == null || state is! SessionSignedIn) return;
    if (!ref.read(appPrefsProvider).biometricEnabled) return;
    if (ref.read(clockProvider)().difference(since) < biometricLockAfter) return;
    ref.read(tokenStoreProvider).lock();
    state = const SessionLocked();
  }

  Future<bool> shouldOfferBiometric() async {
    final prefs = ref.read(appPrefsProvider);
    if (prefs.biometricOffered || prefs.biometricEnabled) return false;
    return ref.read(biometricAuthProvider).isAvailable();
  }

  Future<void> markBiometricOffered() =>
      ref.read(appPrefsProvider).markBiometricOffered();

  /// Asks for the biometric once before turning the gate on, so it is never
  /// enabled on a finger or face that does not work.
  Future<bool> enableBiometric() async {
    final biometric = ref.read(biometricAuthProvider);
    if (!await biometric.isAvailable() || !await biometric.authenticate()) {
      return false;
    }
    await ref.read(appPrefsProvider).setBiometricEnabled(true);
    return true;
  }

  Future<void> disableBiometric() =>
      ref.read(appPrefsProvider).setBiometricEnabled(false);

  Future<void> _rehydrate() async {
    try {
      await ref.read(tokenRefresherProvider).refresh(null);
      state = SessionSignedIn(await _fetchUser());
    } on SessionExpiredError {
      // The expiry signal already moved a revoked session to sign-in with its
      // notice; a session that was never stored lands here without one.
      if (state is! SessionSignedOut) state = const SessionSignedOut();
    } on ApiError {
      state = const SessionUnreachable();
    }
  }

  Future<CurrentUser> _fetchUser() =>
      callApi(() => ref.read(apiProvider).auth.getV1SessionsMe());

  Future<void> _endLocally({String? notice}) async {
    await ref.read(tokenStoreProvider).clear();
    await ref.read(appPrefsProvider).setHasSession(false);
    state = SessionSignedOut(notice: notice);
  }
}
```

- [ ] **Step 5: Rodar o teste e confirmar o verde**

Run: `flutter test test/core/session`
Expected: PASS, todos os testes da pasta (8 da Task 4, 9 da Task 5, 19 deste arquivo).

- [ ] **Step 6: Commit**

```bash
git add lib/core test/core test/support
git commit -m "feat(session): sign in, rehydrate, sign out and gate the session behind biometrics" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 8: Tema, estados de tela e tradução de erro

**Files:**

- Create: `assets/fonts/` (Geist), `lib/ui/theme.dart`, `lib/ui/async_value_view.dart`, `lib/core/http/error_messages.dart`
- Modify: `pubspec.yaml` (fontes)
- Test: `test/core/http/error_messages_test.dart`, `test/ui/async_value_view_test.dart`

**Interfaces:**

- Consumes: `ApiError`, Riverpod `AsyncValue`.
- Produces: `ThemeData buildTheme()`; `AsyncValueView<T>({required AsyncValue<T> value, required Widget Function(T) data, VoidCallback? onRetry})`; `String messageForError(Object error, {Map<int, String> byStatus})`.

- [ ] **Step 1: Trazer a fonte Geist**

```bash
tmp=$(mktemp -d)
(cd "$tmp" && npm pack geist@1.8.1 --silent && tar xzf geist-1.8.1.tgz)
mkdir -p assets/fonts
for weight in Regular Medium SemiBold Bold; do
  cp "$tmp/package/dist/fonts/geist-sans/Geist-$weight.ttf" assets/fonts/
done
cp "$tmp/package/LICENSE.TXT" assets/fonts/OFL.txt 2>/dev/null || cp "$tmp"/package/LICENSE* assets/fonts/OFL.txt
rm -rf "$tmp"
```

Em `pubspec.yaml`, dentro de `flutter:`:

```yaml
  fonts:
    - family: Geist
      fonts:
        - asset: assets/fonts/Geist-Regular.ttf
        - asset: assets/fonts/Geist-Medium.ttf
          weight: 500
        - asset: assets/fonts/Geist-SemiBold.ttf
          weight: 600
        - asset: assets/fonts/Geist-Bold.ttf
          weight: 700
```

- [ ] **Step 2: Escrever os testes que falham**

`test/core/http/error_messages_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/http/api_error.dart';
import 'package:wa_app/core/http/error_messages.dart';

void main() {
  test('uses the wording the caller gave for that status', () {
    expect(
      messageForError(const ApiError(409, message: 'Duplicate.'),
          byStatus: {409: 'Já existe um fornecedor com esse nome.'}),
      'Já existe um fornecedor com esse nome.',
    );
  });

  test('an unreachable server gets its own sentence', () {
    expect(messageForError(const ApiError(0)),
        'Não foi possível falar com o servidor. Verifique a conexão e tente de novo.');
  });

  test('an ended session says so', () {
    expect(messageForError(const SessionExpiredError()),
        'Sua sessão expirou. Entre novamente.');
  });

  test('never shows the English message from the API', () {
    expect(messageForError(const ApiError(500, message: 'Internal Server Error')),
        'Não foi possível concluir a operação. Tente de novo.');
  });

  test('anything that is not an ApiError falls back to the generic sentence', () {
    expect(messageForError(StateError('bug')),
        'Não foi possível concluir a operação. Tente de novo.');
  });
}
```

`test/ui/async_value_view_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/http/api_error.dart';
import 'package:wa_app/ui/async_value_view.dart';

Widget _host(Widget child) => MaterialApp(home: Scaffold(body: child));

void main() {
  testWidgets('shows a progress indicator while loading', (tester) async {
    await tester.pumpWidget(_host(AsyncValueView<String>(
      value: const AsyncLoading(),
      data: Text.new,
    )));

    expect(find.byType(CircularProgressIndicator), findsOneWidget);
  });

  testWidgets('shows the data when it arrives', (tester) async {
    await tester.pumpWidget(_host(AsyncValueView<String>(
      value: const AsyncData('pronto'),
      data: Text.new,
    )));

    expect(find.text('pronto'), findsOneWidget);
  });

  testWidgets('shows the error in Portuguese with a way out', (tester) async {
    var retried = 0;
    await tester.pumpWidget(_host(AsyncValueView<String>(
      value: AsyncError(const ApiError(0), StackTrace.empty),
      data: Text.new,
      onRetry: () => retried++,
    )));

    expect(find.text('Não foi possível falar com o servidor. Verifique a conexão '
        'e tente de novo.'), findsOneWidget);
    await tester.tap(find.text('Tentar de novo'));
    expect(retried, 1);
  });
}
```

- [ ] **Step 3: Rodar os testes e confirmar o vermelho**

Run: `flutter test test/core/http/error_messages_test.dart test/ui`
Expected: FAIL, porque os módulos não existem.

- [ ] **Step 4: Escrever a implementação mínima**

`lib/core/http/error_messages.dart`:

```dart
import 'package:wa_app/core/http/api_error.dart';

const _unreachable =
    'Não foi possível falar com o servidor. Verifique a conexão e tente de novo.';
const _expired = 'Sua sessão expirou. Entre novamente.';
const _fallback = 'Não foi possível concluir a operação. Tente de novo.';

/// The API answers in English and the interface is in Portuguese, so its
/// message never reaches the screen. The caller gives the wording for the
/// statuses it knows the meaning of in its own context; the rest falls back to
/// a sentence that is honest without pretending to explain. Same rule as
/// wa-web/src/lib/form-errors.ts.
String messageForError(Object error, {Map<int, String> byStatus = const {}}) {
  if (error is! ApiError) return _fallback;
  if (error is SessionExpiredError || error is SessionLockedError) return _expired;
  return byStatus[error.status] ?? (error.status == 0 ? _unreachable : _fallback);
}
```

`lib/ui/async_value_view.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:wa_app/core/http/error_messages.dart';

/// Every screen is loading, showing data, or showing an error with a way out.
/// A blank screen or a swallowed error is not a valid state.
class AsyncValueView<T> extends StatelessWidget {
  const AsyncValueView({
    super.key,
    required this.value,
    required this.data,
    this.onRetry,
  });

  final AsyncValue<T> value;
  final Widget Function(T value) data;
  final VoidCallback? onRetry;

  @override
  Widget build(BuildContext context) => switch (value) {
        AsyncData(:final value) => data(value),
        AsyncError(:final error) => Center(
            child: Padding(
              padding: const EdgeInsets.all(24),
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Text(messageForError(error), textAlign: TextAlign.center),
                  if (onRetry != null) ...[
                    const SizedBox(height: 16),
                    OutlinedButton(onPressed: onRetry, child: const Text('Tentar de novo')),
                  ],
                ],
              ),
            ),
          ),
        _ => const Center(child: CircularProgressIndicator()),
      };
}
```

`lib/ui/theme.dart`:

```dart
import 'package:flutter/material.dart';

// The wa-web palette (stock shadcn neutral), converted from OKLCH.
const _primary = Color(0xFF171717);
const _foreground = Color(0xFF0A0A0A);
const _muted = Color(0xFF737373);
const _border = Color(0xFFE5E5E5);
const _destructive = Color(0xFFE7000B);
const _radius = BorderRadius.all(Radius.circular(10));

/// Touch targets of at least 48px: the shop floor uses phones and tablets.
const _minTarget = Size(48, 48);

ThemeData buildTheme() {
  const scheme = ColorScheme.light(
    primary: _primary,
    onPrimary: Colors.white,
    surface: Colors.white,
    onSurface: _foreground,
    onSurfaceVariant: _muted,
    outline: _border,
    outlineVariant: _border,
    error: _destructive,
    onError: Colors.white,
  );
  const shape = RoundedRectangleBorder(borderRadius: _radius);
  return ThemeData(
    useMaterial3: true,
    colorScheme: scheme,
    fontFamily: 'Geist',
    scaffoldBackgroundColor: Colors.white,
    materialTapTargetSize: MaterialTapTargetSize.padded,
    inputDecorationTheme: const InputDecorationTheme(
      border: OutlineInputBorder(borderRadius: _radius),
      enabledBorder: OutlineInputBorder(
          borderRadius: _radius, borderSide: BorderSide(color: _border)),
    ),
    filledButtonTheme: FilledButtonThemeData(
        style: FilledButton.styleFrom(minimumSize: _minTarget, shape: shape)),
    outlinedButtonTheme: OutlinedButtonThemeData(
        style: OutlinedButton.styleFrom(minimumSize: _minTarget, shape: shape)),
    textButtonTheme: TextButtonThemeData(
        style: TextButton.styleFrom(minimumSize: _minTarget, shape: shape)),
    cardTheme: const CardThemeData(
      shape: RoundedRectangleBorder(
          borderRadius: _radius, side: BorderSide(color: _border)),
      elevation: 0,
    ),
  );
}
```

- [ ] **Step 5: Rodar os testes e confirmar o verde**

Run: `flutter test test/core/http/error_messages_test.dart test/ui && flutter analyze`
Expected: PASS, 8 testes; `No issues found!`.

- [ ] **Step 6: Commit**

```bash
git add assets lib/ui lib/core/http/error_messages.dart test/ui test/core/http/error_messages_test.dart pubspec.yaml
git commit -m "feat(ui): add the theme, the three screen states and Portuguese error wording" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 9: Redirecionamento por estado da sessão

**Files:**

- Create: `lib/app/routes.dart`
- Test: `test/app/routes_test.dart`

**Interfaces:**

- Consumes: `SessionState`.
- Produces: `abstract final class Routes` com `splash`, `unavailable`, `unlock`, `login`, `home`, `suppliers`, `changePassword`; `String? redirectFor(SessionState session, Uri location)`.

- [ ] **Step 1: Escrever o teste que falha**

`test/app/routes_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/app/routes.dart';
import 'package:wa_app/core/session/session_state.dart';

import '../support/samples.dart';

String? go(SessionState session, String location) =>
    redirectFor(session, Uri.parse(location));

void main() {
  final signedIn = SessionSignedIn(aCurrentUser());

  test('while booting everything waits on the splash, remembering the target', () {
    expect(go(const SessionBooting(), '/suppliers'), '/splash?from=%2Fsuppliers');
    expect(go(const SessionBooting(), '/splash'), isNull);
  });

  test('a signed-out user goes to sign-in, carrying the target along', () {
    expect(go(const SessionSignedOut(), '/suppliers'), '/login?from=%2Fsuppliers');
    expect(go(const SessionSignedOut(), '/splash?from=%2Fsuppliers'),
        '/login?from=%2Fsuppliers');
    expect(go(const SessionSignedOut(), '/login'), isNull);
  });

  test('the home is not worth remembering', () {
    expect(go(const SessionSignedOut(), '/'), '/login');
  });

  test('a locked session goes to the unlock screen', () {
    expect(go(const SessionLocked(), '/suppliers'), '/unlock?from=%2Fsuppliers');
  });

  test('an unreachable server goes to its screen', () {
    expect(go(const SessionUnreachable(), '/splash'), '/unavailable');
  });

  test('a signed-in user leaves the entry screens for the remembered target', () {
    expect(go(signedIn, '/login?from=%2Fsuppliers'), '/suppliers');
    expect(go(signedIn, '/unlock'), '/');
    expect(go(signedIn, '/splash'), '/');
  });

  test('a signed-in user stays where they are elsewhere', () {
    expect(go(signedIn, '/suppliers'), isNull);
  });

  test('a target that is not an app path is ignored', () {
    expect(go(signedIn, '/login?from=https%3A%2F%2Fevil.example'), '/');
  });
}
```

- [ ] **Step 2: Rodar o teste e confirmar o vermelho**

Run: `flutter test test/app/routes_test.dart`
Expected: FAIL, porque `routes.dart` não existe.

- [ ] **Step 3: Escrever a implementação mínima**

`lib/app/routes.dart`:

```dart
import 'package:wa_app/core/session/session_state.dart';

abstract final class Routes {
  static const splash = '/splash';
  static const unavailable = '/unavailable';
  static const unlock = '/unlock';
  static const login = '/login';
  static const home = '/';
  static const suppliers = '/suppliers';
  static const changePassword = '/change-password';
}

const _entryPaths = {Routes.splash, Routes.unavailable, Routes.unlock, Routes.login};

/// Where the router sends the user for a given session state. Pure, so every
/// branch is tested without a widget tree.
String? redirectFor(SessionState session, Uri location) {
  final target = switch (session) {
    SessionBooting() => Routes.splash,
    SessionUnreachable() => Routes.unavailable,
    SessionLocked() => Routes.unlock,
    SessionSignedOut() => Routes.login,
    SessionSignedIn() => null,
  };

  if (target == null) {
    if (!_entryPaths.contains(location.path)) return null;
    return _from(location) ?? Routes.home;
  }
  if (location.path == target) return null;

  final from = _entryPaths.contains(location.path)
      ? _from(location)
      : _appPath(location.toString());
  return from == null ? target : Uri(path: target, queryParameters: {'from': from}).toString();
}

String? _from(Uri location) => _appPath(location.queryParameters['from']);

/// Only an in-app path is remembered, and the home is not worth it.
String? _appPath(String? path) =>
    path != null && path.startsWith('/') && !path.startsWith('//') && path != Routes.home
        ? path
        : null;
```

- [ ] **Step 4: Rodar o teste e confirmar o verde**

Run: `flutter test test/app/routes_test.dart`
Expected: PASS, 8 testes.

- [ ] **Step 5: Commit**

```bash
git add lib/app/routes.dart test/app/routes_test.dart
git commit -m "feat(app): route each session state to its screen and remember the target" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 10: Tela de entrada

**Files:**

- Create: `lib/features/auth/login_page.dart`
- Test: `test/features/auth/login_page_test.dart`

**Interfaces:**

- Consumes: `sessionProvider`, `messageForError`.
- Produces: `LoginPage`.

- [ ] **Step 1: Escrever o teste que falha**

`test/features/auth/login_page_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/session/session_controller.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/core/session/session_state.dart';
import 'package:wa_app/core/session/token_store.dart';
import 'package:wa_app/features/auth/login_page.dart';

import '../../support/fake_adapter.dart';
import '../../support/fakes.dart';
import '../../support/harness.dart';
import '../../support/samples.dart';

void main() {
  late FakeAdapter adapter;

  setUp(() {
    adapter = FakeAdapter()
      ..on('POST', '/v1/sessions/signin', (_) => jsonResponse(aTokenPair().toJson()))
      ..on('GET', '/v1/sessions/me', (_) => jsonResponse(aCurrentUser().toJson()));
  });

  Future<ProviderContainer> pump(
    WidgetTester tester, {
    InMemorySecureStore? secure,
    Map<String, Object> prefs = const {'wa.installed': true},
  }) async {
    final container = await makeContainer(adapter: adapter, secure: secure, prefs: prefs);
    await container.read(sessionProvider.notifier).boot();
    await tester.pumpWidget(UncontrolledProviderScope(
      container: container,
      child: const MaterialApp(home: LoginPage()),
    ));
    return container;
  }

  Future<void> fillAndSubmit(WidgetTester tester, String user, String password) async {
    await tester.enterText(find.widgetWithText(TextFormField, 'Usuário'), user);
    await tester.enterText(find.widgetWithText(TextFormField, 'Senha'), password);
    await tester.tap(find.widgetWithText(FilledButton, 'Entrar'));
    await tester.pumpAndSettle();
  }

  testWidgets('asks for both fields before calling the API', (tester) async {
    await pump(tester);

    await tester.tap(find.widgetWithText(FilledButton, 'Entrar'));
    await tester.pumpAndSettle();

    expect(find.text('Informe o usuário.'), findsOneWidget);
    expect(find.text('Informe a senha.'), findsOneWidget);
    expect(adapter.requests, isEmpty);
  });

  testWidgets('signs in with what was typed', (tester) async {
    final container = await pump(tester);

    await fillAndSubmit(tester, 'ana', 'secret');

    expect(container.read(sessionProvider), isA<SessionSignedIn>());
    expect(adapter.requests.first.data, {'username': 'ana', 'password': 'secret'});
  });

  testWidgets('says the credentials are wrong on a 401', (tester) async {
    adapter.on('POST', '/v1/sessions/signin',
        (_) => jsonResponse({'message': 'Invalid credentials.'}, status: 401));
    await pump(tester);

    await fillAndSubmit(tester, 'ana', 'wrong');

    expect(find.text('Usuário ou senha inválidos.'), findsOneWidget);
    expect(find.textContaining('Invalid'), findsNothing);
  });

  testWidgets('asks to wait on a 429', (tester) async {
    adapter.on('POST', '/v1/sessions/signin',
        (_) => jsonResponse({'statusCode': 429, 'message': 'Rate limit exceeded'}, status: 429));
    await pump(tester);

    await fillAndSubmit(tester, 'ana', 'x');

    expect(find.text('Muitas tentativas. Espere um minuto e tente de novo.'), findsOneWidget);
  });

  testWidgets('shows why the user was sent here', (tester) async {
    adapter.on('POST', '/v1/sessions/refresh',
        (_) => jsonResponse({'message': 'Invalid refresh token.'}, status: 401));
    await pump(
      tester,
      secure: InMemorySecureStore()..values[TokenStore.refreshTokenKey] = 'revoked',
      prefs: {'wa.installed': true, 'wa.hasSession': true},
    );

    expect(find.text(sessionExpiredNotice), findsOneWidget);
  });
}
```

- [ ] **Step 2: Rodar o teste e confirmar o vermelho**

Run: `flutter test test/features/auth/login_page_test.dart`
Expected: FAIL, porque `login_page.dart` não existe.

- [ ] **Step 3: Escrever a implementação mínima**

`lib/features/auth/login_page.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:wa_app/core/http/error_messages.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/core/session/session_state.dart';

const _signInMessages = {
  401: 'Usuário ou senha inválidos.',
  429: 'Muitas tentativas. Espere um minuto e tente de novo.',
};

class LoginPage extends ConsumerStatefulWidget {
  const LoginPage({super.key});

  @override
  ConsumerState<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends ConsumerState<LoginPage> {
  final _form = GlobalKey<FormState>();
  final _username = TextEditingController();
  final _password = TextEditingController();
  bool _busy = false;
  String? _error;

  @override
  void dispose() {
    _username.dispose();
    _password.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    if (!_form.currentState!.validate()) return;
    setState(() {
      _busy = true;
      _error = null;
    });
    try {
      await ref.read(sessionProvider.notifier).signIn(_username.text.trim(), _password.text);
    } on Object catch (error) {
      if (mounted) setState(() => _error = messageForError(error, byStatus: _signInMessages));
    } finally {
      if (mounted) setState(() => _busy = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    final session = ref.watch(sessionProvider);
    final notice = session is SessionSignedOut ? session.notice : null;
    final theme = Theme.of(context);

    return Scaffold(
      body: SafeArea(
        child: Center(
          child: SingleChildScrollView(
            padding: const EdgeInsets.all(24),
            child: ConstrainedBox(
              constraints: const BoxConstraints(maxWidth: 400),
              child: Form(
                key: _form,
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.stretch,
                  children: [
                    Text('wa-system', style: theme.textTheme.headlineMedium),
                    const SizedBox(height: 24),
                    if (notice != null) ...[
                      Text(notice, style: theme.textTheme.bodyMedium),
                      const SizedBox(height: 16),
                    ],
                    TextFormField(
                      controller: _username,
                      decoration: const InputDecoration(labelText: 'Usuário'),
                      autofillHints: const [AutofillHints.username],
                      textInputAction: TextInputAction.next,
                      validator: (v) =>
                          (v == null || v.trim().isEmpty) ? 'Informe o usuário.' : null,
                    ),
                    const SizedBox(height: 16),
                    TextFormField(
                      controller: _password,
                      decoration: const InputDecoration(labelText: 'Senha'),
                      obscureText: true,
                      autofillHints: const [AutofillHints.password],
                      onFieldSubmitted: (_) => _submit(),
                      validator: (v) => (v == null || v.isEmpty) ? 'Informe a senha.' : null,
                    ),
                    if (_error != null) ...[
                      const SizedBox(height: 16),
                      Text(_error!, style: TextStyle(color: theme.colorScheme.error)),
                    ],
                    const SizedBox(height: 24),
                    FilledButton(
                      onPressed: _busy ? null : _submit,
                      child: const Text('Entrar'),
                    ),
                  ],
                ),
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```

- [ ] **Step 4: Rodar o teste e confirmar o verde**

Run: `flutter test test/features/auth/login_page_test.dart`
Expected: PASS, 5 testes.

- [ ] **Step 5: Commit**

```bash
git add lib/features/auth/login_page.dart test/features/auth/login_page_test.dart
git commit -m "feat(auth): sign in with username and password" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 11: Guarda de permissão e acesso negado

Cobre o risco 3 da spec.

**Files:**

- Create: `lib/features/auth/require_permission.dart`, `lib/features/auth/access_denied_page.dart`
- Test: `test/features/auth/require_permission_test.dart`

**Interfaces:**

- Consumes: `sessionProvider`, `Permission`.
- Produces: `RequirePermission({required Permission permission, required Widget child})`; `AccessDeniedPage`.

- [ ] **Step 1: Escrever o teste que falha**

`test/features/auth/require_permission_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/features/auth/require_permission.dart';

import '../../support/fake_adapter.dart';
import '../../support/harness.dart';
import '../../support/samples.dart';

/// Counts how many times it was built, to prove a denied screen never mounts.
class _Probe extends StatelessWidget {
  const _Probe(this.onBuild);

  final VoidCallback onBuild;

  @override
  Widget build(BuildContext context) {
    onBuild();
    return const Text('conteúdo protegido');
  }
}

void main() {
  Future<void> pump(WidgetTester tester, List<Permission> permissions, VoidCallback onBuild) async {
    final adapter = FakeAdapter()
      ..on('POST', '/v1/sessions/signin', (_) => jsonResponse(aTokenPair().toJson()))
      ..on('GET', '/v1/sessions/me',
          (_) => jsonResponse(aCurrentUser(permissions: permissions).toJson()));
    final container = await makeContainer(adapter: adapter);
    await container.read(sessionProvider.notifier).boot();
    await container.read(sessionProvider.notifier).signIn('ana', 'x');
    await tester.pumpWidget(UncontrolledProviderScope(
      container: container,
      child: MaterialApp(
        home: RequirePermission(
          permission: Permission.catalogRead,
          child: _Probe(onBuild),
        ),
      ),
    ));
  }

  testWidgets('with the permission, the screen mounts', (tester) async {
    var builds = 0;
    await pump(tester, [Permission.catalogRead], () => builds++);

    expect(find.text('conteúdo protegido'), findsOneWidget);
    expect(builds, greaterThan(0));
  });

  testWidgets('without it, the screen never mounts and access is denied',
      (tester) async {
    var builds = 0;
    await pump(tester, [Permission.catalogCreate], () => builds++);

    expect(builds, 0);
    expect(find.text('Acesso negado'), findsOneWidget);
    expect(find.text('Você não tem permissão para ver esta tela.'), findsOneWidget);
  });
}
```

O `_Probe` é o widget que, numa tela real, dispararia a requisição. Se ele não é construído, a requisição não sai.

- [ ] **Step 2: Rodar o teste e confirmar o vermelho**

Run: `flutter test test/features/auth/require_permission_test.dart`
Expected: FAIL, porque os módulos não existem.

- [ ] **Step 3: Escrever a implementação mínima**

`lib/features/auth/require_permission.dart`:

```dart
import 'package:flutter/widgets.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/core/session/session_state.dart';
import 'package:wa_app/features/auth/access_denied_page.dart';

/// Does not build [child] without the permission, so a denied screen never
/// mounts and never calls the API. The API stays the authority.
class RequirePermission extends ConsumerWidget {
  const RequirePermission({super.key, required this.permission, required this.child});

  final Permission permission;
  final Widget child;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final session = ref.watch(sessionProvider);
    final allowed = session is SessionSignedIn && session.can(permission);
    return allowed ? child : const AccessDeniedPage();
  }
}
```

`lib/features/auth/access_denied_page.dart`:

```dart
import 'package:flutter/material.dart';

/// A 403 is an expected state, not a defect: no error styling, no bug report.
class AccessDeniedPage extends StatelessWidget {
  const AccessDeniedPage({super.key});

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Text('Acesso negado', style: theme.textTheme.titleLarge),
            const SizedBox(height: 8),
            const Text('Você não tem permissão para ver esta tela.', textAlign: TextAlign.center),
            const SizedBox(height: 4),
            Text(
              'Fale com quem administra o acesso se precisar dela.',
              textAlign: TextAlign.center,
              style: TextStyle(color: theme.colorScheme.onSurfaceVariant),
            ),
          ],
        ),
      ),
    );
  }
}
```

- [ ] **Step 4: Rodar o teste e confirmar o verde**

Run: `flutter test test/features/auth/require_permission_test.dart`
Expected: PASS, 2 testes.

- [ ] **Step 5: Commit**

```bash
git add lib/features/auth test/features/auth/require_permission_test.dart
git commit -m "feat(auth): keep a screen unmounted when the user lacks its permission" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 12: Desbloqueio e servidor indisponível

**Files:**

- Create: `lib/features/auth/unlock_page.dart`, `lib/features/auth/unavailable_page.dart`, `lib/app/splash_page.dart`
- Test: `test/features/auth/unlock_page_test.dart`, `test/features/auth/unavailable_page_test.dart`

**Interfaces:**

- Consumes: `sessionProvider`.
- Produces: `UnlockPage`, `UnavailablePage`, `SplashPage`.

- [ ] **Step 1: Escrever os testes que falham**

`test/features/auth/unlock_page_test.dart`:

```dart
import 'dart:async';

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/core/session/session_state.dart';
import 'package:wa_app/core/session/token_store.dart';
import 'package:wa_app/features/auth/unlock_page.dart';

import '../../support/fake_adapter.dart';
import '../../support/fakes.dart';
import '../../support/harness.dart';
import '../../support/samples.dart';

void main() {
  late FakeBiometricAuth biometric;

  Future<ProviderContainer> pump(WidgetTester tester) async {
    final adapter = FakeAdapter()
      ..on('POST', '/v1/sessions/refresh', (_) => jsonResponse(aTokenPair().toJson()))
      ..on('GET', '/v1/sessions/me', (_) => jsonResponse(aCurrentUser().toJson()));
    final secure = InMemorySecureStore()..values[TokenStore.refreshTokenKey] = 'r';
    final container = await makeContainer(
      adapter: adapter,
      secure: secure,
      biometric: biometric,
      prefs: {'wa.installed': true, 'wa.hasSession': true, 'wa.biometricEnabled': true},
    );
    await container.read(sessionProvider.notifier).boot();
    await tester.pumpWidget(UncontrolledProviderScope(
      container: container,
      child: const MaterialApp(home: UnlockPage()),
    ));
    await tester.pumpAndSettle();
    return container;
  }

  setUp(() => biometric = FakeBiometricAuth());

  testWidgets('prompts for the biometric as soon as it opens', (tester) async {
    final container = await pump(tester);

    expect(biometric.prompts, 1);
    expect(container.read(sessionProvider), isA<SessionSignedIn>());
  });

  testWidgets('the password is always a way out', (tester) async {
    biometric.pending = Completer<bool>();
    final container = await pump(tester);
    expect(container.read(sessionProvider), isA<SessionLocked>());

    await tester.tap(find.text('Entrar com senha'));
    await tester.pump();

    expect(container.read(sessionProvider), isA<SessionSignedOut>());
  });
}
```

`test/features/auth/unavailable_page_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/core/session/session_state.dart';
import 'package:wa_app/core/session/token_store.dart';
import 'package:wa_app/features/auth/unavailable_page.dart';

import '../../support/fake_adapter.dart';
import '../../support/fakes.dart';
import '../../support/harness.dart';
import '../../support/samples.dart';

void main() {
  testWidgets('explains the server is down and retries on demand', (tester) async {
    final adapter = FakeAdapter()..on('POST', '/v1/sessions/refresh', unreachable);
    final secure = InMemorySecureStore()..values[TokenStore.refreshTokenKey] = 'r';
    final container = await makeContainer(
      adapter: adapter,
      secure: secure,
      prefs: {'wa.installed': true, 'wa.hasSession': true},
    );
    await container.read(sessionProvider.notifier).boot();
    await tester.pumpWidget(UncontrolledProviderScope(
      container: container,
      child: const MaterialApp(home: UnavailablePage()),
    ));

    expect(find.text('Não foi possível falar com o servidor. Verifique a conexão '
        'e tente de novo.'), findsOneWidget);

    adapter
      ..on('POST', '/v1/sessions/refresh', (_) => jsonResponse(aTokenPair().toJson()))
      ..on('GET', '/v1/sessions/me', (_) => jsonResponse(aCurrentUser().toJson()));
    await tester.tap(find.text('Tentar de novo'));
    await tester.pumpAndSettle();

    expect(container.read(sessionProvider), isA<SessionSignedIn>());
  });
}
```

- [ ] **Step 2: Rodar os testes e confirmar o vermelho**

Run: `flutter test test/features/auth/unlock_page_test.dart test/features/auth/unavailable_page_test.dart`
Expected: FAIL, porque os módulos não existem.

- [ ] **Step 3: Escrever a implementação mínima**

`lib/features/auth/unlock_page.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:wa_app/core/session/session_providers.dart';

class UnlockPage extends ConsumerStatefulWidget {
  const UnlockPage({super.key});

  @override
  ConsumerState<UnlockPage> createState() => _UnlockPageState();
}

class _UnlockPageState extends ConsumerState<UnlockPage> {
  @override
  void initState() {
    super.initState();
    // Prompt right away: the user opened the app to use it.
    WidgetsBinding.instance.addPostFrameCallback((_) => _unlock());
  }

  Future<void> _unlock() => ref.read(sessionProvider.notifier).unlock();

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return Scaffold(
      body: SafeArea(
        child: Center(
          child: Padding(
            padding: const EdgeInsets.all(24),
            child: Column(
              mainAxisSize: MainAxisSize.min,
              crossAxisAlignment: CrossAxisAlignment.stretch,
              children: [
                Text('Sessão travada',
                    style: theme.textTheme.titleLarge, textAlign: TextAlign.center),
                const SizedBox(height: 8),
                const Text('Use a digital ou o rosto para continuar.',
                    textAlign: TextAlign.center),
                const SizedBox(height: 24),
                FilledButton(onPressed: _unlock, child: const Text('Desbloquear')),
                const SizedBox(height: 8),
                TextButton(
                  onPressed: ref.read(sessionProvider.notifier).usePassword,
                  child: const Text('Entrar com senha'),
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}
```

`lib/features/auth/unavailable_page.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:wa_app/core/http/api_error.dart';
import 'package:wa_app/core/http/error_messages.dart';
import 'package:wa_app/core/session/session_providers.dart';

class UnavailablePage extends ConsumerWidget {
  const UnavailablePage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) => Scaffold(
        body: SafeArea(
          child: Center(
            child: Padding(
              padding: const EdgeInsets.all(24),
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Text(messageForError(const ApiError(0)), textAlign: TextAlign.center),
                  const SizedBox(height: 16),
                  FilledButton(
                    onPressed: ref.read(sessionProvider.notifier).retry,
                    child: const Text('Tentar de novo'),
                  ),
                ],
              ),
            ),
          ),
        ),
      );
}
```

`lib/app/splash_page.dart`:

```dart
import 'package:flutter/material.dart';

class SplashPage extends StatelessWidget {
  const SplashPage({super.key});

  @override
  Widget build(BuildContext context) =>
      const Scaffold(body: Center(child: CircularProgressIndicator()));
}
```

- [ ] **Step 4: Rodar os testes e confirmar o verde**

Run: `flutter test test/features/auth`
Expected: PASS, todos os testes da pasta.

- [ ] **Step 5: Commit**

```bash
git add lib/features/auth lib/app/splash_page.dart test/features/auth
git commit -m "feat(auth): unlock with biometrics and recover when the server is down at startup" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 13: Troca da própria senha

**Files:**

- Create: `lib/features/auth/change_password_page.dart`
- Test: `test/features/auth/change_password_page_test.dart`

**Interfaces:**

- Consumes: `sessionProvider.changePassword`, `messageForError`.
- Produces: `ChangePasswordPage`.

- [ ] **Step 1: Escrever o teste que falha**

`test/features/auth/change_password_page_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/session/session_controller.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/core/session/session_state.dart';
import 'package:wa_app/features/auth/change_password_page.dart';

import '../../support/fake_adapter.dart';
import '../../support/harness.dart';
import '../../support/samples.dart';

void main() {
  late FakeAdapter adapter;

  setUp(() {
    adapter = FakeAdapter()
      ..on('POST', '/v1/sessions/signin', (_) => jsonResponse(aTokenPair().toJson()))
      ..on('POST', '/v1/sessions/refresh',
          (_) => jsonResponse(aTokenPair(access: 'a2', refresh: 'r2').toJson()))
      ..on('GET', '/v1/sessions/me', (_) => jsonResponse(aCurrentUser().toJson()))
      ..on('PATCH', '/v1/sessions/me/password', (_) => emptyResponse());
  });

  Future<ProviderContainer> pump(WidgetTester tester) async {
    final container = await makeContainer(adapter: adapter);
    await container.read(sessionProvider.notifier).boot();
    await container.read(sessionProvider.notifier).signIn('ana', 'x');
    await tester.pumpWidget(UncontrolledProviderScope(
      container: container,
      child: const MaterialApp(home: Scaffold(body: ChangePasswordPage())),
    ));
    return container;
  }

  Future<void> fill(WidgetTester tester, String current, String next, String repeat) async {
    await tester.enterText(find.widgetWithText(TextFormField, 'Senha atual'), current);
    await tester.enterText(find.widgetWithText(TextFormField, 'Nova senha'), next);
    await tester.enterText(find.widgetWithText(TextFormField, 'Repita a nova senha'), repeat);
    await tester.tap(find.widgetWithText(FilledButton, 'Trocar a senha'));
    await tester.pumpAndSettle();
  }

  testWidgets('validates before calling the API', (tester) async {
    await pump(tester);

    await fill(tester, '', 'short', 'other');

    expect(find.text('Informe a senha atual.'), findsOneWidget);
    expect(find.text('A nova senha precisa de pelo menos 8 caracteres.'), findsOneWidget);
    expect(find.text('As senhas não conferem.'), findsOneWidget);
    expect(adapter.count('PATCH', '/v1/sessions/me/password'), 0);
  });

  testWidgets('a changed password ends the session with its notice', (tester) async {
    final container = await pump(tester);

    await fill(tester, 'x', 'new-secret', 'new-secret');

    expect(container.read(sessionProvider), isA<SessionSignedOut>()
        .having((s) => s.notice, 'notice', passwordChangedNotice));
    expect(adapter.requests.last.data,
        {'currentPassword': 'x', 'newPassword': 'new-secret'});
  });

  testWidgets('a wrong current password says so and keeps the session', (tester) async {
    adapter.on('PATCH', '/v1/sessions/me/password',
        (_) => jsonResponse({'message': 'Invalid credentials.'}, status: 401));
    final container = await pump(tester);

    await fill(tester, 'wrong', 'new-secret', 'new-secret');

    expect(find.text('A senha atual não confere.'), findsOneWidget);
    expect(container.read(sessionProvider), isA<SessionSignedIn>());
  });
}
```

- [ ] **Step 2: Rodar o teste e confirmar o vermelho**

Run: `flutter test test/features/auth/change_password_page_test.dart`
Expected: FAIL, porque `change_password_page.dart` não existe.

- [ ] **Step 3: Escrever a implementação mínima**

`lib/features/auth/change_password_page.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:wa_app/core/http/api_error.dart';
import 'package:wa_app/core/http/error_messages.dart';
import 'package:wa_app/core/session/session_providers.dart';

const _minLength = 8;

class ChangePasswordPage extends ConsumerStatefulWidget {
  const ChangePasswordPage({super.key});

  @override
  ConsumerState<ChangePasswordPage> createState() => _ChangePasswordPageState();
}

class _ChangePasswordPageState extends ConsumerState<ChangePasswordPage> {
  final _form = GlobalKey<FormState>();
  final _current = TextEditingController();
  final _next = TextEditingController();
  final _repeat = TextEditingController();
  bool _busy = false;
  String? _currentError;
  String? _error;

  @override
  void dispose() {
    _current.dispose();
    _next.dispose();
    _repeat.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    setState(() {
      _currentError = null;
      _error = null;
    });
    if (!_form.currentState!.validate()) return;
    setState(() => _busy = true);
    try {
      await ref.read(sessionProvider.notifier).changePassword(_current.text, _next.text);
    } on ApiError catch (error) {
      if (!mounted) return;
      setState(() {
        if (error.status == 401 && error is! SessionExpiredError) {
          _currentError = 'A senha atual não confere.';
        } else {
          _error = messageForError(error);
        }
      });
    } finally {
      if (mounted) setState(() => _busy = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return SingleChildScrollView(
      padding: const EdgeInsets.all(24),
      child: Form(
        key: _form,
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            TextFormField(
              controller: _current,
              obscureText: true,
              decoration: InputDecoration(labelText: 'Senha atual', errorText: _currentError),
              validator: (v) => (v == null || v.isEmpty) ? 'Informe a senha atual.' : null,
            ),
            const SizedBox(height: 16),
            TextFormField(
              controller: _next,
              obscureText: true,
              decoration: const InputDecoration(labelText: 'Nova senha'),
              validator: (v) => (v == null || v.length < _minLength)
                  ? 'A nova senha precisa de pelo menos 8 caracteres.'
                  : null,
            ),
            const SizedBox(height: 16),
            TextFormField(
              controller: _repeat,
              obscureText: true,
              decoration: const InputDecoration(labelText: 'Repita a nova senha'),
              validator: (v) {
                if (v == null || v.isEmpty) return 'Repita a nova senha.';
                return v == _next.text ? null : 'As senhas não conferem.';
              },
            ),
            if (_error != null) ...[
              const SizedBox(height: 16),
              Text(_error!, style: TextStyle(color: theme.colorScheme.error)),
            ],
            const SizedBox(height: 24),
            FilledButton(
              onPressed: _busy ? null : _submit,
              child: const Text('Trocar a senha'),
            ),
          ],
        ),
      ),
    );
  }
}
```

- [ ] **Step 4: Rodar o teste e confirmar o verde**

Run: `flutter test test/features/auth/change_password_page_test.dart`
Expected: PASS, 3 testes.

- [ ] **Step 5: Commit**

```bash
git add lib/features/auth/change_password_page.dart test/features/auth/change_password_page_test.dart
git commit -m "feat(auth): change your own password and sign in again with it" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 14: Fornecedores: acesso à API e lista

**Files:**

- Create: `lib/features/suppliers/suppliers_repository.dart`, `lib/features/suppliers/suppliers_page.dart`
- Test: `test/features/suppliers/suppliers_page_test.dart`

**Interfaces:**

- Consumes: `apiProvider`, `callApi`, `AsyncValueView`, `sessionProvider`.
- Produces: `suppliersRepositoryProvider` (`SuppliersRepository` com `list()` e `create(String name)`), `suppliersProvider` (`FutureProvider.autoDispose<List<Supplier>>`), `SuppliersPage`.

- [ ] **Step 1: Escrever o teste que falha**

`test/features/suppliers/suppliers_page_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/features/suppliers/suppliers_page.dart';

import '../../support/fake_adapter.dart';
import '../../support/harness.dart';
import '../../support/samples.dart';

void main() {
  late FakeAdapter adapter;

  Future<void> pump(WidgetTester tester, {List<Permission> permissions = const [
    Permission.catalogRead,
  ]}) async {
    adapter
      ..on('POST', '/v1/sessions/signin', (_) => jsonResponse(aTokenPair().toJson()))
      ..on('GET', '/v1/sessions/me',
          (_) => jsonResponse(aCurrentUser(permissions: permissions).toJson()));
    final container = await makeContainer(adapter: adapter);
    await container.read(sessionProvider.notifier).boot();
    await container.read(sessionProvider.notifier).signIn('ana', 'x');
    await tester.pumpWidget(UncontrolledProviderScope(
      container: container,
      child: const MaterialApp(home: SuppliersPage()),
    ));
    await tester.pumpAndSettle();
  }

  setUp(() => adapter = FakeAdapter());

  testWidgets('lists the suppliers with a count', (tester) async {
    adapter.on('GET', '/v1/suppliers', (_) => jsonResponse([
          aSupplier(name: 'Moinho Santa Clara', id: '0199a8a0-0000-7000-8000-00000000000a').toJson(),
          aSupplier(name: 'Laticínio Boa Vista', id: '0199a8a0-0000-7000-8000-00000000000b').toJson(),
        ]));
    await pump(tester);

    expect(find.text('Moinho Santa Clara'), findsOneWidget);
    expect(find.text('Laticínio Boa Vista'), findsOneWidget);
    expect(find.text('2 fornecedores'), findsOneWidget);
  });

  testWidgets('says so when there is none yet', (tester) async {
    adapter.on('GET', '/v1/suppliers', (_) => jsonResponse([]));
    await pump(tester);

    expect(find.text('Nenhum fornecedor cadastrado ainda.'), findsOneWidget);
  });

  testWidgets('offers to retry when the list fails', (tester) async {
    adapter.on('GET', '/v1/suppliers', unreachable);
    await pump(tester);

    expect(find.text('Tentar de novo'), findsOneWidget);

    adapter.on('GET', '/v1/suppliers', (_) => jsonResponse([aSupplier().toJson()]));
    await tester.tap(find.text('Tentar de novo'));
    await tester.pumpAndSettle();
    expect(find.text('1 fornecedor'), findsOneWidget);
  });

  testWidgets('the create button needs CATALOG_CREATE', (tester) async {
    adapter.on('GET', '/v1/suppliers', (_) => jsonResponse([]));
    await pump(tester);

    expect(find.text('Novo fornecedor'), findsNothing);
  });

  testWidgets('with CATALOG_CREATE the create button is there', (tester) async {
    adapter.on('GET', '/v1/suppliers', (_) => jsonResponse([]));
    await pump(tester, permissions: [Permission.catalogRead, Permission.catalogCreate]);

    expect(find.text('Novo fornecedor'), findsOneWidget);
  });
}
```

- [ ] **Step 2: Rodar o teste e confirmar o vermelho**

Run: `flutter test test/features/suppliers/suppliers_page_test.dart`
Expected: FAIL, porque os módulos não existem.

- [ ] **Step 3: Escrever a implementação mínima**

`lib/features/suppliers/suppliers_repository.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/api/api_providers.dart';
import 'package:wa_app/core/http/api_error.dart';

class SuppliersRepository {
  SuppliersRepository(this._api);

  final WaApi _api;

  Future<List<Supplier>> list() => callApi(_api.suppliers.getV1Suppliers);

  Future<void> create(String name) =>
      callApi(() => _api.suppliers.postV1Suppliers(body: CreateSupplierBody(name: name)));
}

final suppliersRepositoryProvider =
    Provider<SuppliersRepository>((ref) => SuppliersRepository(ref.watch(apiProvider)));

final suppliersProvider = FutureProvider.autoDispose<List<Supplier>>(
  (ref) => ref.watch(suppliersRepositoryProvider).list(),
);
```

`lib/features/suppliers/suppliers_page.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/core/session/session_state.dart';
import 'package:wa_app/features/suppliers/suppliers_repository.dart';
import 'package:wa_app/ui/async_value_view.dart';

class SuppliersPage extends ConsumerWidget {
  const SuppliersPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final session = ref.watch(sessionProvider);
    final canCreate = session is SessionSignedIn && session.can(Permission.catalogCreate);

    return Scaffold(
      floatingActionButton: canCreate
          ? FloatingActionButton.extended(
              onPressed: () {},
              icon: const Icon(Icons.add),
              label: const Text('Novo fornecedor'),
            )
          : null,
      body: AsyncValueView(
        value: ref.watch(suppliersProvider),
        onRetry: () => ref.invalidate(suppliersProvider),
        data: (suppliers) => suppliers.isEmpty
            ? const Center(child: Text('Nenhum fornecedor cadastrado ainda.'))
            : _SupplierList(suppliers),
      ),
    );
  }
}

class _SupplierList extends StatelessWidget {
  const _SupplierList(this.suppliers);

  final List<Supplier> suppliers;

  @override
  Widget build(BuildContext context) {
    final count = suppliers.length;
    return ListView.separated(
      padding: const EdgeInsets.only(bottom: 96),
      itemCount: count + 1,
      separatorBuilder: (_, index) => index == 0 ? const SizedBox.shrink() : const Divider(height: 1),
      itemBuilder: (context, index) {
        if (index == 0) {
          return Padding(
            padding: const EdgeInsets.fromLTRB(16, 16, 16, 8),
            child: Text(
              count == 1 ? '1 fornecedor' : '$count fornecedores',
              style: TextStyle(color: Theme.of(context).colorScheme.onSurfaceVariant),
            ),
          );
        }
        return ListTile(title: Text(suppliers[index - 1].name));
      },
    );
  }
}
```

O `onPressed` vazio do botão é preenchido na Task 15.

- [ ] **Step 4: Rodar o teste e confirmar o verde**

Run: `flutter test test/features/suppliers/suppliers_page_test.dart`
Expected: PASS, 5 testes.

- [ ] **Step 5: Commit**

```bash
git add lib/features/suppliers test/features/suppliers
git commit -m "feat(suppliers): list the suppliers" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 15: Fornecedores: cadastro

**Files:**

- Create: `lib/features/suppliers/supplier_form_sheet.dart`
- Modify: `lib/features/suppliers/suppliers_page.dart`
- Test: `test/features/suppliers/supplier_form_sheet_test.dart`

**Interfaces:**

- Consumes: `suppliersRepositoryProvider`, `suppliersProvider`, `messageForError`.
- Produces: `Future<void> showSupplierFormSheet(BuildContext context)`.

- [ ] **Step 1: Escrever o teste que falha**

`test/features/suppliers/supplier_form_sheet_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/features/suppliers/suppliers_page.dart';

import '../../support/fake_adapter.dart';
import '../../support/harness.dart';
import '../../support/samples.dart';

void main() {
  late FakeAdapter adapter;
  late List<Supplier> stored;

  setUp(() {
    stored = [];
    adapter = FakeAdapter()
      ..on('POST', '/v1/sessions/signin', (_) => jsonResponse(aTokenPair().toJson()))
      ..on('GET', '/v1/sessions/me', (_) => jsonResponse(aCurrentUser(
          permissions: [Permission.catalogRead, Permission.catalogCreate]).toJson()))
      ..on('GET', '/v1/suppliers',
          (_) => jsonResponse(stored.map((s) => s.toJson()).toList()))
      ..on('POST', '/v1/suppliers', (request) {
        final name = (request.data as Map<String, Object?>)['name']! as String;
        final created = aSupplier(name: name);
        stored.add(created);
        return jsonResponse(created.toJson(), status: 201);
      });
  });

  Future<void> pumpAndOpen(WidgetTester tester) async {
    final container = await makeContainer(adapter: adapter);
    await container.read(sessionProvider.notifier).boot();
    await container.read(sessionProvider.notifier).signIn('ana', 'x');
    await tester.pumpWidget(UncontrolledProviderScope(
      container: container,
      child: const MaterialApp(home: SuppliersPage()),
    ));
    await tester.pumpAndSettle();
    await tester.tap(find.text('Novo fornecedor'));
    await tester.pumpAndSettle();
  }

  Future<void> submit(WidgetTester tester, String name) async {
    await tester.enterText(find.widgetWithText(TextFormField, 'Nome'), name);
    await tester.tap(find.widgetWithText(FilledButton, 'Cadastrar'));
    await tester.pumpAndSettle();
  }

  testWidgets('creates the supplier, closes, confirms and refreshes the list',
      (tester) async {
    await pumpAndOpen(tester);

    await submit(tester, '  Moinho Santa Clara  ');

    expect(adapter.requests.lastWhere((r) => r.method == 'POST').data,
        {'name': 'Moinho Santa Clara'});
    expect(find.widgetWithText(TextFormField, 'Nome'), findsNothing);
    expect(find.text('Fornecedor cadastrado.'), findsOneWidget);
    expect(find.text('Moinho Santa Clara'), findsOneWidget);
  });

  testWidgets('asks for a name before calling the API', (tester) async {
    await pumpAndOpen(tester);

    await submit(tester, '   ');

    expect(find.text('Informe o nome do fornecedor.'), findsOneWidget);
    expect(adapter.count('POST', '/v1/suppliers'), 0);
  });

  testWidgets('refuses a name longer than 120 characters', (tester) async {
    await pumpAndOpen(tester);

    await submit(tester, 'x' * 121);

    expect(find.text('O nome pode ter no máximo 120 caracteres.'), findsOneWidget);
    expect(adapter.count('POST', '/v1/suppliers'), 0);
  });

  testWidgets('a duplicate name is reported on the field', (tester) async {
    adapter.on('POST', '/v1/suppliers', (_) => jsonResponse(
        {'message': 'A supplier with this name already exists.'}, status: 409));
    await pumpAndOpen(tester);

    await submit(tester, 'Moinho Santa Clara');

    expect(find.text('Já existe um fornecedor com esse nome.'), findsOneWidget);
    expect(find.widgetWithText(TextFormField, 'Nome'), findsOneWidget);
  });
}
```

- [ ] **Step 2: Rodar o teste e confirmar o vermelho**

Run: `flutter test test/features/suppliers/supplier_form_sheet_test.dart`
Expected: FAIL, porque tocar em "Novo fornecedor" ainda não abre nada.

- [ ] **Step 3: Escrever a implementação mínima**

`lib/features/suppliers/supplier_form_sheet.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:wa_app/core/http/api_error.dart';
import 'package:wa_app/core/http/error_messages.dart';
import 'package:wa_app/features/suppliers/suppliers_repository.dart';

const _maxNameLength = 120;

Future<void> showSupplierFormSheet(BuildContext context) async {
  final created = await showModalBottomSheet<bool>(
    context: context,
    isScrollControlled: true,
    showDragHandle: true,
    builder: (_) => const _SupplierFormSheet(),
  );
  if (created == true && context.mounted) {
    ScaffoldMessenger.of(context)
        .showSnackBar(const SnackBar(content: Text('Fornecedor cadastrado.')));
  }
}

class _SupplierFormSheet extends ConsumerStatefulWidget {
  const _SupplierFormSheet();

  @override
  ConsumerState<_SupplierFormSheet> createState() => _SupplierFormSheetState();
}

class _SupplierFormSheetState extends ConsumerState<_SupplierFormSheet> {
  final _form = GlobalKey<FormState>();
  final _name = TextEditingController();
  bool _busy = false;
  String? _nameError;
  String? _error;

  @override
  void dispose() {
    _name.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    setState(() {
      _nameError = null;
      _error = null;
    });
    if (!_form.currentState!.validate()) return;
    setState(() => _busy = true);
    try {
      await ref.read(suppliersRepositoryProvider).create(_name.text.trim());
      ref.invalidate(suppliersProvider);
      if (mounted) Navigator.of(context).pop(true);
    } on ApiError catch (error) {
      if (!mounted) return;
      setState(() {
        if (error.status == 409) {
          _nameError = 'Já existe um fornecedor com esse nome.';
        } else {
          _error = messageForError(error);
        }
      });
    } finally {
      if (mounted) setState(() => _busy = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return Padding(
      padding: EdgeInsets.fromLTRB(
          24, 0, 24, 24 + MediaQuery.viewInsetsOf(context).bottom),
      child: Form(
        key: _form,
        child: Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            Text('Novo fornecedor', style: theme.textTheme.titleLarge),
            const SizedBox(height: 16),
            TextFormField(
              controller: _name,
              autofocus: true,
              decoration: InputDecoration(labelText: 'Nome', errorText: _nameError),
              textCapitalization: TextCapitalization.words,
              onFieldSubmitted: (_) => _submit(),
              validator: (v) {
                final name = v?.trim() ?? '';
                if (name.isEmpty) return 'Informe o nome do fornecedor.';
                if (name.length > _maxNameLength) {
                  return 'O nome pode ter no máximo 120 caracteres.';
                }
                return null;
              },
            ),
            if (_error != null) ...[
              const SizedBox(height: 16),
              Text(_error!, style: TextStyle(color: theme.colorScheme.error)),
            ],
            const SizedBox(height: 24),
            FilledButton(
              onPressed: _busy ? null : _submit,
              child: const Text('Cadastrar'),
            ),
          ],
        ),
      ),
    );
  }
}
```

Em `lib/features/suppliers/suppliers_page.dart`, importar `supplier_form_sheet.dart` e trocar `onPressed: () {}` por `onPressed: () => showSupplierFormSheet(context)`.

- [ ] **Step 4: Rodar o teste e confirmar o verde**

Run: `flutter test test/features/suppliers`
Expected: PASS, 9 testes.

- [ ] **Step 5: Commit**

```bash
git add lib/features/suppliers test/features/suppliers
git commit -m "feat(suppliers): register a supplier" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 16: Roteador, casco e app

**Files:**

- Create: `lib/app/nav_items.dart`, `lib/app/app_shell.dart`, `lib/app/home_page.dart`, `lib/app/router.dart`, `lib/app/app.dart`
- Modify: `lib/main.dart`
- Test: `test/app/app_test.dart`, `test/app/nav_items_test.dart`

**Interfaces:**

- Consumes: tudo das tarefas anteriores.
- Produces: `NavItem` e `visibleNavItems(SessionSignedIn)`; `AppShell`; `HomePage`; `routerProvider`; `WaApp`; `main()`.

- [ ] **Step 1: Escrever os testes que falham**

`test/app/nav_items_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/app/nav_items.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/session/session_state.dart';

import '../support/samples.dart';

List<String> labels(List<Permission> permissions) =>
    visibleNavItems(SessionSignedIn(aCurrentUser(permissions: permissions)))
        .map((item) => item.label)
        .toList();

void main() {
  test('without permissions only the open destinations show', () {
    expect(labels([]), ['Início', 'Trocar a senha']);
  });

  test('CATALOG_READ shows the suppliers', () {
    expect(labels([Permission.catalogRead]), ['Início', 'Fornecedores', 'Trocar a senha']);
  });
}
```

`test/app/app_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:wa_app/app/app.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/session/token_store.dart';

import '../support/fake_adapter.dart';
import '../support/fakes.dart';
import '../support/harness.dart';
import '../support/samples.dart';

void main() {
  late FakeAdapter adapter;

  setUp(() {
    adapter = FakeAdapter()
      ..on('POST', '/v1/sessions/signin', (_) => jsonResponse(aTokenPair().toJson()))
      ..on('POST', '/v1/sessions/refresh', (_) => jsonResponse(aTokenPair().toJson()))
      ..on('POST', '/v1/sessions/signout', (_) => emptyResponse())
      ..on('GET', '/v1/sessions/me', (_) => jsonResponse(
          aCurrentUser(name: 'Ana', permissions: [Permission.catalogRead]).toJson()))
      ..on('GET', '/v1/suppliers', (_) => jsonResponse([aSupplier().toJson()]));
  });

  Future<void> pumpApp(WidgetTester tester, ProviderContainer container) async {
    await tester.pumpWidget(UncontrolledProviderScope(container: container, child: const WaApp()));
    await tester.pumpAndSettle();
  }

  testWidgets('a first start lands on sign-in, and signing in lands on home',
      (tester) async {
    final biometric = FakeBiometricAuth()..available = false;
    await pumpApp(tester, await makeContainer(adapter: adapter, biometric: biometric));

    expect(find.widgetWithText(FilledButton, 'Entrar'), findsOneWidget);

    await tester.enterText(find.widgetWithText(TextFormField, 'Usuário'), 'ana');
    await tester.enterText(find.widgetWithText(TextFormField, 'Senha'), 'secret');
    await tester.tap(find.widgetWithText(FilledButton, 'Entrar'));
    await tester.pumpAndSettle();

    expect(find.text('Bem-vindo, Ana'), findsOneWidget);
  });

  testWidgets('a stored session opens straight on home', (tester) async {
    final secure = InMemorySecureStore()..values[TokenStore.refreshTokenKey] = 'r';
    await pumpApp(tester, await makeContainer(
      adapter: adapter,
      secure: secure,
      prefs: {'wa.installed': true, 'wa.hasSession': true},
    ));

    expect(find.text('Bem-vindo, Ana'), findsOneWidget);
  });

  testWidgets('the drawer reaches the suppliers and signs out', (tester) async {
    final secure = InMemorySecureStore()..values[TokenStore.refreshTokenKey] = 'r';
    await pumpApp(tester, await makeContainer(
      adapter: adapter,
      secure: secure,
      prefs: {'wa.installed': true, 'wa.hasSession': true, 'wa.biometricOffered': true},
    ));

    await tester.tap(find.byTooltip('Abrir menu de navegação'));
    await tester.pumpAndSettle();
    await tester.tap(find.text('Fornecedores'));
    await tester.pumpAndSettle();
    expect(find.text('Moinho Santa Clara'), findsOneWidget);

    await tester.tap(find.byTooltip('Abrir menu de navegação'));
    await tester.pumpAndSettle();
    await tester.tap(find.text('Sair'));
    await tester.pumpAndSettle();
    expect(find.widgetWithText(FilledButton, 'Entrar'), findsOneWidget);
    expect(adapter.count('POST', '/v1/sessions/signout'), 1);
  });

  testWidgets('offers the biometric gate once after signing in', (tester) async {
    final biometric = FakeBiometricAuth();
    final container = await makeContainer(adapter: adapter, biometric: biometric);
    await pumpApp(tester, container);
    await tester.enterText(find.widgetWithText(TextFormField, 'Usuário'), 'ana');
    await tester.enterText(find.widgetWithText(TextFormField, 'Senha'), 'secret');
    await tester.tap(find.widgetWithText(FilledButton, 'Entrar'));
    await tester.pumpAndSettle();

    expect(find.text('Entrar com biometria?'), findsOneWidget);
    await tester.tap(find.text('Ativar'));
    await tester.pumpAndSettle();
    expect(biometric.prompts, 1);
  });
}
```

O texto do botão do menu (`Abrir menu de navegação`) é o tooltip padrão do Material em português. O app usa `pt_BR` (passo 3), e o teste depende disso.

- [ ] **Step 2: Rodar os testes e confirmar o vermelho**

Run: `flutter test test/app`
Expected: FAIL, porque `nav_items.dart` e `app.dart` não existem.

- [ ] **Step 3: Escrever a implementação mínima**

Adicionar a localização do Material em português:

```bash
flutter pub add flutter_localizations --sdk=flutter
```

`lib/app/nav_items.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:wa_app/app/routes.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/session/session_state.dart';

class NavItem {
  const NavItem(this.label, this.icon, this.route, [this.permission]);

  final String label;
  final IconData icon;
  final String route;

  /// Null means every signed-in user may go there.
  final Permission? permission;
}

const navItems = [
  NavItem('Início', Icons.home_outlined, Routes.home),
  NavItem('Fornecedores', Icons.local_shipping_outlined, Routes.suppliers,
      Permission.catalogRead),
  NavItem('Trocar a senha', Icons.password_outlined, Routes.changePassword),
];

List<NavItem> visibleNavItems(SessionSignedIn session) => [
      for (final item in navItems)
        if (item.permission == null || session.can(item.permission!)) item,
    ];
```

`lib/app/home_page.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/core/session/session_state.dart';

class HomePage extends ConsumerWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final session = ref.watch(sessionProvider);
    final name = session is SessionSignedIn ? session.user.name : '';
    return Center(
      child: Text('Bem-vindo, $name', style: Theme.of(context).textTheme.headlineSmall),
    );
  }
}
```

`lib/app/app_shell.dart`:

```dart
import 'dart:async';

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:wa_app/app/nav_items.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/core/session/session_state.dart';

class AppShell extends ConsumerStatefulWidget {
  const AppShell({super.key, required this.location, required this.child});

  final String location;
  final Widget child;

  @override
  ConsumerState<AppShell> createState() => _AppShellState();
}

class _AppShellState extends ConsumerState<AppShell> {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addPostFrameCallback((_) => _offerBiometric());
  }

  Future<void> _offerBiometric() async {
    final controller = ref.read(sessionProvider.notifier);
    if (!await controller.shouldOfferBiometric() || !mounted) return;
    await controller.markBiometricOffered();
    if (!mounted) return;
    final accepted = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Entrar com biometria?'),
        content: const Text(
            'Da próxima vez, use a digital ou o rosto para abrir o wa-system.'),
        actions: [
          TextButton(
              onPressed: () => Navigator.pop(context, false),
              child: const Text('Agora não')),
          FilledButton(
              onPressed: () => Navigator.pop(context, true), child: const Text('Ativar')),
        ],
      ),
    );
    if (accepted == true) await controller.enableBiometric();
  }

  @override
  Widget build(BuildContext context) {
    final session = ref.watch(sessionProvider);
    if (session is! SessionSignedIn) return const SizedBox.shrink();
    final items = visibleNavItems(session);
    final current = items.where((i) => i.route == widget.location).firstOrNull;

    return Scaffold(
      appBar: AppBar(title: Text(current?.label ?? 'wa-system')),
      drawer: NavigationDrawer(
        selectedIndex: current == null ? null : items.indexOf(current),
        onDestinationSelected: (index) {
          Navigator.pop(context);
          context.go(items[index].route);
        },
        children: [
          Padding(
            padding: const EdgeInsets.fromLTRB(28, 16, 16, 16),
            child: Text(session.user.name, style: Theme.of(context).textTheme.titleMedium),
          ),
          for (final item in items)
            NavigationDrawerDestination(icon: Icon(item.icon), label: Text(item.label)),
          const Divider(indent: 28, endIndent: 28),
          const _BiometricSwitch(),
          ListTile(
            contentPadding: const EdgeInsets.symmetric(horizontal: 28),
            leading: const Icon(Icons.logout),
            title: const Text('Sair'),
            onTap: () {
              Navigator.pop(context);
              unawaited(ref.read(sessionProvider.notifier).signOut());
            },
          ),
        ],
      ),
      body: SafeArea(child: widget.child),
    );
  }
}

class _BiometricSwitch extends ConsumerStatefulWidget {
  const _BiometricSwitch();

  @override
  ConsumerState<_BiometricSwitch> createState() => _BiometricSwitchState();
}

class _BiometricSwitchState extends ConsumerState<_BiometricSwitch> {
  @override
  Widget build(BuildContext context) {
    final available = ref.watch(biometricAvailableProvider).value ?? false;
    if (!available) return const SizedBox.shrink();
    final enabled = ref.read(appPrefsProvider).biometricEnabled;
    return SwitchListTile(
      contentPadding: const EdgeInsets.symmetric(horizontal: 28),
      title: const Text('Entrar com biometria'),
      value: enabled,
      onChanged: (on) async {
        final controller = ref.read(sessionProvider.notifier);
        on ? await controller.enableBiometric() : await controller.disableBiometric();
        if (mounted) setState(() {});
      },
    );
  }
}
```

`lib/app/router.dart`:

```dart
import 'package:flutter/foundation.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:wa_app/app/app_shell.dart';
import 'package:wa_app/app/home_page.dart';
import 'package:wa_app/app/routes.dart';
import 'package:wa_app/app/splash_page.dart';
import 'package:wa_app/core/api/api.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/core/session/session_state.dart';
import 'package:wa_app/features/auth/change_password_page.dart';
import 'package:wa_app/features/auth/login_page.dart';
import 'package:wa_app/features/auth/require_permission.dart';
import 'package:wa_app/features/auth/unavailable_page.dart';
import 'package:wa_app/features/auth/unlock_page.dart';
import 'package:wa_app/features/suppliers/suppliers_page.dart';

final routerProvider = Provider<GoRouter>((ref) {
  final session = ValueNotifier<SessionState>(ref.read(sessionProvider));
  ref.listen(sessionProvider, (_, next) => session.value = next);
  ref.onDispose(session.dispose);

  final router = GoRouter(
    initialLocation: Routes.splash,
    refreshListenable: session,
    redirect: (_, state) => redirectFor(session.value, state.uri),
    routes: [
      GoRoute(path: Routes.splash, builder: (_, _) => const SplashPage()),
      GoRoute(path: Routes.unavailable, builder: (_, _) => const UnavailablePage()),
      GoRoute(path: Routes.unlock, builder: (_, _) => const UnlockPage()),
      GoRoute(path: Routes.login, builder: (_, _) => const LoginPage()),
      ShellRoute(
        builder: (_, state, child) => AppShell(location: state.uri.path, child: child),
        routes: [
          GoRoute(path: Routes.home, builder: (_, _) => const HomePage()),
          GoRoute(
            path: Routes.suppliers,
            builder: (_, _) => const RequirePermission(
              permission: Permission.catalogRead,
              child: SuppliersPage(),
            ),
          ),
          GoRoute(path: Routes.changePassword, builder: (_, _) => const ChangePasswordPage()),
        ],
      ),
    ],
  );
  ref.onDispose(router.dispose);
  return router;
});
```

`lib/app/app.dart`:

```dart
import 'dart:async';

import 'package:flutter/material.dart';
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:wa_app/app/router.dart';
import 'package:wa_app/core/session/session_providers.dart';
import 'package:wa_app/ui/theme.dart';

class WaApp extends ConsumerStatefulWidget {
  const WaApp({super.key});

  @override
  ConsumerState<WaApp> createState() => _WaAppState();
}

class _WaAppState extends ConsumerState<WaApp> {
  late final AppLifecycleListener _lifecycle;

  @override
  void initState() {
    super.initState();
    final controller = ref.read(sessionProvider.notifier);
    _lifecycle = AppLifecycleListener(
      onHide: controller.onBackgrounded,
      onShow: controller.onForegrounded,
    );
    unawaited(controller.boot());
  }

  @override
  void dispose() {
    _lifecycle.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => MaterialApp.router(
        title: 'wa-system',
        theme: buildTheme(),
        routerConfig: ref.watch(routerProvider),
        locale: const Locale('pt', 'BR'),
        supportedLocales: const [Locale('pt', 'BR')],
        localizationsDelegates: GlobalMaterialLocalizations.delegates,
      );
}
```

`lib/main.dart`:

```dart
import 'package:flutter/widgets.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:wa_app/app/app.dart';
import 'package:wa_app/core/env/env.dart';
import 'package:wa_app/core/http/http_providers.dart';
import 'package:wa_app/core/http/retry_policy.dart';
import 'package:wa_app/core/session/session_providers.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final env = Env.fromEnvironment();
  final sharedPreferences = await SharedPreferences.getInstance();
  runApp(ProviderScope(
    retry: retryPolicy,
    overrides: [
      envProvider.overrideWithValue(env),
      sharedPreferencesProvider.overrideWithValue(sharedPreferences),
    ],
    child: const WaApp(),
  ));
}
```

- [ ] **Step 4: Rodar os testes e confirmar o verde**

Run: `flutter test test/app`
Expected: PASS, todos os testes da pasta (8 de `routes_test`, 2 de `nav_items_test`, 4 de `app_test`).

- [ ] **Step 5: Commit**

```bash
git add lib test pubspec.yaml pubspec.lock
git commit -m "feat(app): wire the router, the permission-filtered drawer and the app entry point" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Task 17: Configuração nativa

Sem teste automatizado: o que muda é manifesto e plist. A verificação é o build e a conferência manual no aparelho, e o relatório da tarefa diz isso.

**Files:**

- Modify: `android/app/src/main/AndroidManifest.xml`, `android/app/src/debug/AndroidManifest.xml`, `android/app/src/main/kotlin/br/com/hipersenna/wa_app/MainActivity.kt`
- Modify: `ios/Runner/Info.plist`

- [ ] **Step 1: Android**

Em `android/app/src/main/AndroidManifest.xml`, antes de `<application>`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.USE_BIOMETRIC" />
```

O template do Flutter só declara `INTERNET` nos manifestos de debug e profile; sem a linha acima, o build de release não acessa a rede. No mesmo arquivo, trocar o `android:label` para `wa-system`.

Em `android/app/src/debug/AndroidManifest.xml`, dentro de `<manifest>`:

```xml
<application android:usesCleartextTraffic="true" />
```

HTTP puro só no build de debug, para falar com `http://localhost:3333` pelo `adb reverse`. O build de release continua exigindo HTTPS.

Em `MainActivity.kt`, trocar a classe base, exigência do `local_auth`:

```kotlin
package br.com.hipersenna.wa_app

import io.flutter.embedding.android.FlutterFragmentActivity

class MainActivity : FlutterFragmentActivity()
```

- [ ] **Step 2: iOS**

Em `ios/Runner/Info.plist`, dentro do `<dict>` principal:

```xml
<key>CFBundleDisplayName</key>
<string>wa-system</string>
<key>NSFaceIDUsageDescription</key>
<string>Usamos o Face ID para desbloquear sua sessão no wa-system.</string>
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSAllowsLocalNetworking</key>
  <true/>
</dict>
```

Se `CFBundleDisplayName` já existir, só trocar o valor. O iOS não compila no WSL; esta parte fica sem verificação até haver macOS, e o relatório da tarefa diz isso.

- [ ] **Step 3: Verificar o build Android**

Run: `flutter build apk --debug --dart-define=API_BASE_URL=http://localhost:3333`
Expected: `✓ Built build/app/outputs/flutter-apk/app-debug.apk`.

- [ ] **Step 4: Commit**

```bash
git add android ios
git commit -m "chore(app): allow biometrics, release networking and local HTTP in debug" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fechamento da fatia

- [ ] **Verificação final**

Run: `scripts/check.sh`
Expected: análise sem aviso, formatação em dia, suíte inteira verde. Cole a saída; sem ela, nenhuma afirmação de pronto.

- [ ] **Conferência manual, no aparelho, com a API no ar**

Com o `wa-api` rodando na máquina, o celular repassado ao WSL pelo `usbipd` e `scripts/dev.sh`:

1. Primeira abertura: tela de entrada. Entrar com um usuário que tenha `CATALOG_READ` e `CATALOG_CREATE`.
2. O app oferece a biometria. Ativar e confirmar a digital.
3. Fechar o app de vez e abrir de novo: pede a digital antes de mostrar qualquer dado. Confirmar: entra direto na tela inicial.
4. Mandar o app para o segundo plano por mais de 5 minutos e voltar: pede a digital de novo.
5. Na tela travada, tocar em "Entrar com senha": vai para a tela de entrada.
6. Abrir Fornecedores, cadastrar um, tentar cadastrar outro com o mesmo nome (deve dizer que já existe).
7. Entrar com um usuário sem `CATALOG_READ`: a gaveta não mostra "Fornecedores".
8. Parar a API e abrir o app: tela de servidor indisponível. Subir a API e tocar em "Tentar de novo": entra.
9. Trocar a própria senha: a sessão termina e a tela de entrada mostra "Senha trocada. Entre de novo com a senha nova."
10. Entrar no app e no `wa-web` com o mesmo usuário, usar os dois por mais de 15 minutos (o tempo de vida do access token): nenhum dos dois cai para o login.

- [ ] **Abrir o PR** (somente sob ordem explícita do dono do projeto)

```
## What changed
- Flutter project with a validated environment and strict analysis — the base every slice builds on
- API contract and HTTP clients generated from the wa-api OpenAPI document — a contract change breaks the build
- HTTP client with a single-flight token refresh — stops the API's chain revocation from logging operators out
- session with the refresh token in the system keystore and rehydration at startup — the session survives closing the app
- optional biometric unlock that gates the stored session — a phone left unlocked does not open the system
- routing by session state and permission guard — a denied screen never mounts nor calls the API
- permission-filtered drawer, sign-in, unlock, server-down and change-password screens — the shell around every slice
- theme on the wa-web palette with the Geist font — the app and the web look like one system
- suppliers list and registration — the slice that proves the foundation end to end
- Android biometric, network and debug cleartext settings, and iOS Face ID text — what the native side needs to run it

## Dependencies
- flutter_riverpod@3.4.3 — server state and dependency wiring
- go_router@18.1.0 — routing and redirects by session state
- dio@5.11.1, retrofit@4.10.0, json_annotation@4.12.0 — HTTP and generated clients
- flutter_secure_storage@11.2.0 — refresh token in Keychain and Keystore
- local_auth@3.0.2 — biometric unlock
- shared_preferences@2.5.6 — install marker and non-secret flags
- decimal@3.2.6, intl@0.20.3 — decimal and pt-BR formatting for the next slices
- flutter_localizations (Flutter SDK) — Material texts in Portuguese
- swagger_parser@1.45.0, build_runner@2.16.2, json_serializable@6.15.1, retrofit_generator@10.2.12 — contract generation
- mocktail@1.0.5, flutter_lints — tests and lints
- husky@9.1.7, @commitlint/cli@21.2.2, @commitlint/config-conventional@21.2.2 — commit convention
```

Antes de abrir, conferir as versões contra o `pubspec.lock` e o `package-lock.json` e corrigir a lista.

## Riscos de execução deste plano

- **O código gerado não foi o deste plano.** Os nomes de cliente e de método (`postV1SessionsRefresh`, `getV1Suppliers`) e de modelo (`GetV1SessionsMeResponse`) vêm do teste de geração contra `52aa0e1`. Se o `openapi.json` mudou desde então, o erro aparece em `api.dart` e nas chamadas; conserte os apelidos ali, nunca o gerado.
- **Construtores gerados podem não ser `const`.** O teste da Task 2 usa `const` nos corpos de requisição. Se o gerador não emitir construtores `const`, retirar o `const` dos testes.
- **Timers pendentes em teste de widget.** O `dio` pode deixar timers de timeout ativos quando a resposta falsa volta. Se um `testWidgets` falhar com "A Timer is still pending", zerar os timeouts no harness (um provider de opções do `dio` substituído em teste) em vez de afrouxar o teste.
- **`local_auth` no Android pede tema AppCompat em versões antigas.** Se a biometria travar em Android 9 ou anterior, trocar o `LaunchTheme` para um tema `Theme.AppCompat`. O aparelho de teste é Android 15.
- **`adb` sem permissão.** Se o udev não estiver ativo no WSL, o `scripts/dev.sh` falha no `adb reverse` com `no permissions`. O contorno é `adb kill-server && sudo $(which adb) start-server` antes de rodar o script.
- **iOS sem verificação.** A Task 17 altera o `Info.plist` sem poder compilar. A primeira compilação em macOS pode pedir ajustes, que entram como correção própria.
