# Guia de Estilo da Documentação - Colibri Project Go

Este guia define os padrões de escrita e formatação para a documentação do `colibri-sdk-go`.

## 1. Tom e Voz

*   **Tom**: Prestativo, técnico e acessível.
*   **Voz**: 
    *   Utilize a **primeira pessoa do plural** ("Nós", "Configuramos", "Vejamos") para tutoriais e guias passo a passo, criando uma sensação de acompanhamento.
    *   Utilize a **terceira pessoa** ("O pacote fornece", "A função executa") para descrições técnicas e definições de componentes.
*   **Linguagem**: Português do Brasil (PT-BR).

## 2. Terminologia Padronizada

Para manter a consistência, utilize sempre os termos abaixo:

*   **Microsserviço** (evite: microserviço, micro-serviço).
*   **Framework**.
*   **Endpoint**.
*   **Middleware**.
*   **Query**.
*   **Statement**.
*   **Cache**.
*   **Observabilidade**.
*   **Roteamento**.
*   **Payload**.
*   **Header**.
*   **SDK**.

## 3. Formatação Markdown

*   **Títulos**:
    *   `# Título` (apenas via frontmatter `title`).
    *   `## Seção Principal` (H2).
    *   `### Subseção` (H3).
*   **Blocos de Código**:
    *   Sempre especifique a linguagem (ex: `go`, `shell`, `dotenv`, `sql`).
    *   Utilize `showLineNumbers` para blocos com mais de 5 linhas.
*   **Termos Técnicos**: Utilize crases para nomes de arquivos, funções, variáveis e pacotes (ex: `main.go`, `InitializeApp()`, `colibri-sdk-go`).
*   **Destaque**: Utilize **negrito** para termos importantes ou ênfase moderada.
*   **Links**: Links para bibliotecas externas devem ser feitos na primeira menção.
*   **Separador**: Finalize todos os arquivos com `___`.

## 4. Estrutura de Páginas de Funcionalidades (Features)

Siga esta ordem preferencial:

1.  **Resumo/Introdução**: O que é a funcionalidade e para que serve.
2.  **Configuração (Variáveis de Ambiente)**: Quais configurações são necessárias.
3.  **Inicialização**: Como ativar a funcionalidade no `main.go`.
4.  **Componentes Principais**: Detalhamento técnico.
5.  **Exemplos de Uso**: Código prático.
6.  **Boas Práticas** (Opcional).

___
