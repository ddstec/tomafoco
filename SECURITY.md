# Política de segurança

## Versões suportadas

Só a versão mais recente recebe correções. O app se atualiza sozinho pelo Sparkle; quem desativou a checagem automática pode atualizar em **Configurações › Atualizações › Buscar atualizações…**.

## Como reportar uma vulnerabilidade

**Não abra issue pública.** Use um destes canais privados:

- **GitHub:** aba *Security* do repositório › *Report a vulnerability* (aviso de segurança privado).
- **E-mail:** contato@dds.tec.br, com "[segurança] Tomafoco" no assunto.

Inclua a versão do Tomafoco e do macOS, os passos para reproduzir e o impacto que você enxerga. Confirmamos o recebimento em até 7 dias e combinamos com você a data de divulgação depois da correção.

## O que nos interessa especialmente

- **Cadeia de atualização (Sparkle):** qualquer forma de fazer o app aceitar um update que não foi assinado com a chave EdDSA do projeto.
- **Automação de navegador:** injeção de AppleScript a partir de domínios ou URLs configurados pelo usuário.
- **Senha de desbloqueio:** contornar a senha guardada no Keychain para liberar um app ou desligar o bloqueio contínuo.
- Execução de código ou escalonamento de privilégio a partir do Tomafoco.

Um usuário que fecha o Tomafoco à força, desinstala o app ou revoga a permissão de Automação para escapar do bloqueio **não** é vulnerabilidade: o app ajuda a manter o foco e não foi feito para resistir a quem administra o próprio Mac.
