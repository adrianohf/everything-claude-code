---
name: bun-runtime
description: Bun como runtime, gerenciador de pacotes, bundler e executor de testes (test runner). Quando escolher Bun vs Node, notas de migração e suporte ao Vercel.
origin: ECC
---

# Bun Runtime

O Bun é um runtime de JavaScript rápido "tudo-em-um" e um conjunto de ferramentas: runtime, gerenciador de pacotes, bundler e executor de testes.

## Quando Usar

- **Prefira o Bun** para: novos projetos JS/TS, scripts onde a velocidade de instalação/execução é importante, implantações (deployments) no Vercel com runtime Bun e quando você deseja uma única cadeia de ferramentas (run + install + test + build).
- **Prefira o Node** para: compatibilidade máxima com o ecossistema, ferramentas legadas que assumem o Node ou quando uma dependência tem problemas conhecidos com o Bun.

Use ao: adotar o Bun, migrar do Node, escrever ou depurar scripts/testes do Bun, ou configurar o Bun no Vercel ou em outras plataformas.

## Como Funciona

- **Runtime**: Runtime compatível com Node e de fácil substituição (construído sobre JavaScriptCore, implementado em Zig).
- **Gerenciador de pacotes**: O `bun install` é significativamente mais rápido que o npm/yarn. O arquivo de bloqueio é `bun.lock` (texto) por padrão no Bun atual; versões anteriores usavam `bun.lockb` (binário).
- **Bundler**: Bundler e transpilador integrados para aplicativos e bibliotecas.
- **Executor de testes**: `bun test` integrado com API semelhante ao Jest.

**Migração do Node**: Substitua `node script.js` por `bun run script.js` ou `bun script.js`. Execute `bun install` no lugar de `npm install`; a maioria dos pacotes funciona. Use `bun run` para scripts npm; `bun x` para execuções únicas no estilo npx. Os módulos integrados do Node são suportados; prefira as APIs do Bun onde elas existirem para melhor desempenho.

**Vercel**: Defina o runtime como Bun nas configurações do projeto. Build: `bun run build` ou `bun build ./src/index.ts --outdir=dist`. Instalação: `bun install --frozen-lockfile` para implantações reproduzíveis.

## Exemplos

### Execução e instalação

```bash
# Instalar dependências (cria/atualiza bun.lock ou bun.lockb)
bun install

# Executar um script ou arquivo
bun run dev
bun run src/index.ts
bun src/index.ts
```

### Scripts e ambiente (env)

```bash
bun run --env-file=.env dev
FOO=bar bun run script.ts
```

### Testes

```bash
bun test
bun test --watch
```

```typescript
// test/example.test.ts
import { expect, test } from "bun:test";

test("add", () => {
  expect(1 + 2).toBe(3);
});
```

### API do Runtime

```typescript
const file = Bun.file("package.json");
const json = await file.json();

Bun.serve({
  port: 3000,
  fetch(req) {
    return new Response("Hello");
  },
});
```

## Melhores Práticas

- Faça o commit do arquivo de bloqueio (`bun.lock` ou `bun.lockb`) para instalações reproduzíveis.
- Prefira `bun run` para scripts. Para TypeScript, o Bun executa `.ts` nativamente.
- Mantenha as dependências atualizadas; o Bun e o ecossistema evoluem rapidamente.
