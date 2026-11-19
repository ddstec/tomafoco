# Como contribuir

Obrigado pelo interesse no Tomafoco! Issues, correções e ideias são bem-vindas. O projeto é mantido por uma pessoa no tempo livre, então respostas podem levar alguns dias.

## Antes de começar

- **Bug?** Abra uma issue com o modelo *Bug*: versão do Tomafoco, versão do macOS, navegador (se for bloqueio de site) e passos para reproduzir.
- **Funcionalidade nova?** Abra uma issue antes de escrever código. Assim combinamos o escopo e você não perde trabalho num PR que não cabe no produto.
- **Vulnerabilidade?** Não abra issue — siga o [SECURITY.md](SECURITY.md).

## Ambiente

Veja *Compilar a partir do código* no [README](README.md). Resumo:

```bash
brew install xcodegen swiftlint
make open    # gera e abre o projeto
make test    # testes dos pacotes SPM
make lint    # SwiftLint
```

Não é preciso conta Apple Developer: sem `Config/Local.xcconfig`, o app é assinado só para rodar na sua máquina.

## Regras de arquitetura

O projeto segue Clean Architecture com pacotes SPM por camada ([docs/arquitetura.md](docs/arquitetura.md)). Um PR que quebre estas regras volta para ajuste:

- **TomafocoDomain** não importa AppKit, SwiftUI nem nada do macOS, e não depende de nenhum outro pacote.
- **TomafocoApplication** depende só do Domain.
- **TomafocoInfrastructure** implementa os ports do Domain; nunca depende da Application.
- Nada de `Date()` ou `Timer` no Domain/Application — use o `SessionClock`.
- Efeito colateral sai da `SessionStateMachine` como `SessionEffect`; quem executa é o `SessionCoordinator`.
- A UI recebe casos de uso por injeção pelo `AppContainer`; View não fala direto com adapter de infraestrutura.

## Pull requests

1. Faça fork e crie uma branch a partir da `main` (`feat/…`, `fix/…`, `docs/…`).
2. Escreva testes para regra nova ou bug corrigido — no Domain e na Application, teste de unidade é obrigatório.
3. Rode `make test` e `make lint` antes de abrir o PR. A CI roda os testes e compila o app.
4. Mantenha o PR focado em um assunto. Refatoração grande e mudança de comportamento vão em PRs separados.
5. Mudança de interface? Anexe uma captura de tela clara e uma escura.
6. **Não mexa** em `MARKETING_VERSION`, `CURRENT_PROJECT_VERSION`, `SUFeedURL` nem `SUPublicEDKey` — release é com o mantenedor.

### Commits

Mensagens em português, no estilo [Conventional Commits](https://www.conventionalcommits.org/pt-br/):

```
feat: importa subtarefas do Lembretes
fix: mantém o bloqueio após o Mac acordar do sleep
docs: explica a permissão de Automação no README
```

## Idioma e estilo

- Interface, mensagens e documentação em **português do Brasil**.
- Nomes no código (tipos, funções, variáveis) em **inglês**, como no restante do projeto.
- Siga o `.swiftlint.yml`: sem `force_unwrapping`, `force_cast` ou `force_try` em produção; funções curtas.

## Licença das contribuições

Ao enviar uma contribuição, você concorda que ela seja licenciada sob a mesma licença do projeto, a [GNU AGPL-3.0 ou posterior](LICENSE), e declara que tem o direito de enviá-la.

## Conduta

Todas as interações seguem o [Código de Conduta](CODE_OF_CONDUCT.md).
