# Marca e identidade visual

O código-fonte do Tomafoco é livre sob a [AGPL-3.0](LICENSE). A licença cobre o **código**, não a **marca**: como prevê a seção 7(e) da AGPL, ela não concede direito de uso de nomes comerciais, marcas ou logotipos.

## O que é da marca

- O nome **Tomafoco**.
- O ícone do app (o tomate dentro do anel) e o símbolo usado na barra de menus.
- A identidade visual e o nome **DDS.TEC**.
- O domínio `tomafoco.dds.tec.br` e o canal oficial de atualizações (`appcast.xml`).

## Pode, sem pedir

- Compilar e usar o Tomafoco para você mesmo, com o nome e o ícone originais.
- Fazer fork, estudar, modificar e enviar pull requests.
- Citar o nome "Tomafoco" para dizer de onde veio o seu projeto ("baseado no Tomafoco").

## Se for distribuir uma versão modificada

Para ninguém confundir o seu build com o oficial:

1. **Troque o nome** do app (`name`, `PRODUCT_NAME` e `CFBundleDisplayName` no `project.yml`).
2. **Troque o ícone** (`App/Resources/Assets.xcassets/AppIcon.appiconset`) e o símbolo da barra de menus.
3. **Troque o `PRODUCT_BUNDLE_IDENTIFIER`** — com o mesmo identificador, o seu app divide permissões, Keychain e preferências com o oficial.
4. **Troque ou remova `SUFeedURL` e `SUPublicEDKey`.** Com os valores originais, o seu build checa o feed oficial e oferece aos seus usuários a substituição pelo Tomafoco oficial.
5. Não sugira que a sua versão é oficial ou endossada pela DDS.TEC.

Dúvidas ou pedidos de uso da marca: **contato@dds.tec.br**.
