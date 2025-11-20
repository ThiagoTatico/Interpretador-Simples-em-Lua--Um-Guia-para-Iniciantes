# Construindo um Interpretador Simples em Lua: Um Guia para Iniciantes

Este tutorial é um guia passo a passo para construir um interpretador simples para a linguagem ficticia **Uninter**, usando a linguagem de programação **Lua**. O objetivo é desmistificar o funcionamento interno de interpretadores, tornando o processo acessível a iniciantes em programação.

O interpretador que construiremos não fará a análise léxica (Lexing) nem a análise sintática (Parsing). Em vez disso, ele receberá diretamente a **Árvore de Sintaxe Abstrata (AST)** da linguagem Uninter, que é um objeto JSON que representa a estrutura do código-fonte.

## 1. Conceitos Fundamentais: A Árvore de Sintaxe Abstrata (AST)

Um interpretador tradicional tem três fases principais:

1.  **Análise Léxica (Lexing):** Transforma o código-fonte em uma sequência de *tokens* (palavras-chave, identificadores, números, etc.).
2.  **Análise Sintática (Parsing):** Organiza os *tokens* em uma estrutura hierárquica chamada **Árvore de Sintaxe Abstrata (AST)**.
3.  **Interpretação/Execução:** Percorre a AST e executa as ações correspondentes a cada nó.

Nosso interpretador começará na **Fase 3**. A AST que receberemos é um objeto JSON, onde cada nó (ou expressão) é representado por uma tabela com um campo `kind` (tipo) que define a operação a ser realizada.

**Exemplo de AST (JSON):**

```json
{
  "kind": "Print",
  "value": {
    "kind": "Str",
    "value": "Hello World"
  }
}
```

Este JSON representa o código que imprime a string "Hello World". Nosso interpretador irá ler o `kind: "Print"`, e então interpretar o `value` (que é a string "Hello World") e, por fim, executar a ação de impressão.

## 2. Configuração do Ambiente

Você precisará apenas do **Lua 5.3+** instalado.

1.  **Instale o Lua:** Se você não tem o Lua instalado, siga as instruções para o seu sistema operacional.
2.  **Módulo JSON:** Nosso interpretador precisa ler o arquivo JSON da AST. O arquivo `json.lua` fornecido é um módulo de terceiros que implementa a decodificação de JSON em Lua.
    *   Certifique-se de que o arquivo `json.lua` esteja no mesmo diretório do seu interpretador principal.

## 3. O Coração do Interpretador: A Função `interpret`

O interpretador será uma única função recursiva que percorre a AST: `interpret(ast_node, context)`.

*   `ast_node`: O nó atual da AST (uma tabela Lua).
*   `context`: Uma tabela que armazena as variáveis e funções definidas (o **escopo**).

### 3.1. Estrutura Base (`main.lua`)

Crie um arquivo chamado `main.lua` e adicione a estrutura básica:

```lua
-- main.lua

-- Importa o módulo JSON para decodificar a AST
json = require "json"

-- A função principal do interpretador
function interpret(ast_node, context)
    -- Verifica se o nó é uma tabela (expressão válida)
    if type(ast_node) ~= "table" then
        return ast_node
    end
    
    -- O tipo de operação a ser realizada
    local kind = ast_node["kind"]
    -- O contexto (escopo) de variáveis
    local variables = context or {}

    -- Se não houver um 'kind', retorna o próprio nó (útil para valores literais)
    if kind == nil then
        return ast_node
    end

    -- Lógica de interpretação para cada 'kind' (a ser implementada)
    -- ...

    -- Se o 'kind' for desconhecido, reporta um erro
    if kind ~= nil then
        print("Unknown command: " .. kind)
        os.exit()
    end
end

-- Lógica para carregar o arquivo JSON e iniciar a interpretação
local input_file = arg[1] or "source.rinha.json"

local file = io.open(input_file, "r")
if file == nil then
    print("Could not read \"" .. input_file .. "\"")
    os.exit()
end

local content = file:read "*a"
file:close()

-- A AST da Uninter é uma lista de expressões (embora geralmente seja apenas uma)
local ast = json.decode(content)

-- Inicia a interpretação com um contexto vazio
interpret(ast["expression"], {})
```

### 3.2. Implementando os Tipos de Dados e `Print`

Os tipos de dados básicos (`Str`, `Int`, `Bool`) são os mais simples, pois apenas retornam seus valores.

O `Print` é o primeiro comando de execução que usaremos.

```lua
-- Adicione dentro da função interpret, substituindo o comentário '-- ...'

    if kind == "Str" then
        -- Returns the string value
        return ast_node["value"]
    elseif kind == "Int" then
        -- Returns the integer value
        return ast_node["value"]
    elseif kind == "Bool" then
        -- Returns the boolean value
        return ast_node["value"]
    elseif kind == "Print" then
        -- Interprets the value to be printed
        local value = interpret(ast_node["value"], variables)
        -- Prints the value (Lua's print handles basic types)
        print(tostring(value))
        -- Returns the value (expressions in Uninter always return a value)
        return value
    -- ... outros 'kind' virão aqui
```

### 3.3. Implementando Variáveis: `Let` e `Var`

A gestão de variáveis é feita através da tabela `variables` (o `context`).

*   `Let`: Define uma nova variável no escopo.
*   `Var`: Recupera o valor de uma variável do escopo.

```lua
-- Adicione dentro da função interpret

    elseif kind == "Let" then
        -- 1. Interpret the value of the expression
        local value = interpret(ast_node["value"], variables)
        -- 2. Store the value in the context using the variable name
        variables[ast_node["name"]["text"]] = value
        -- 3. Continue interpreting the next expression in the sequence
        return interpret(ast_node["next"], variables)
    elseif kind == "Var" then
        -- 1. Retrieve the value from the context using the variable name
        local value = variables[ast_node["text"]]
        -- 2. If the variable is not found, it's an error (optional check)
        if value == nil then
            print("Error: Variable '" .. ast_node["text"] .. "' not found.")
            os.exit()
        end
        -- 3. Return the variable's value
        return value
```

### 3.4. Implementando Operações Binárias: `Binary`

O `Binary` lida com operações matemáticas, lógicas e de comparação.

```lua
-- Adicione dentro da função interpret

    elseif kind == "Binary" then
        local operator = ast_node["op"]
        -- Interpret the left-hand side (lhs) and right-hand side (rhs)
        local lhs = interpret(ast_node["lhs"], variables)
        local rhs = interpret(ast_node["rhs"], variables)
        
        -- Lógica para cada operador
        if operator == "Add" then
            -- Handles string concatenation or number addition
            if type(lhs) == "string" or type(rhs) == "string" then
                return lhs .. rhs
            else
                return lhs + rhs
            end
        elseif operator == "Sub" then
            return lhs - rhs
        elseif operator == "Mul" then
            return lhs * rhs
        elseif operator == "Div" then
            -- Basic division with floor for integer result
            if rhs == 0 then
                print("Division by zero error.")
                os.exit()
            end
            return math.floor(lhs / rhs)
        elseif operator == "Rem" then
            return lhs % rhs
        elseif operator == "Eq" then
            return lhs == rhs
        elseif operator == "Neq" then
            return lhs ~= rhs
        elseif operator == "Lt" then
            return lhs < rhs
        elseif operator == "Lte" then
            return lhs <= rhs
        elseif operator == "Gt" then
            return lhs > rhs
        elseif operator == "Gte" then
            return lhs >= rhs
        elseif operator == "And" then
            return lhs and rhs
        elseif operator == "Or" then
            return lhs or rhs
        else
            print("Unknown operator: " .. operator)
            os.exit()
        end
```

### 3.5. Implementando Estruturas de Controle: `If`

O `If` é simples: interpreta a condição e, dependendo do resultado (verdadeiro ou falso), interpreta o bloco `then` ou o bloco `otherwise`.

```lua
-- Adicione dentro da função interpret

    elseif kind == "If" then
        -- Interprets the condition
        local condition_result = interpret(ast_node["condition"], variables)
        
        -- Lua trata 'nil' e 'false' como falso. Qualquer outra coisa é verdadeiro.
        if condition_result then
            -- Interprets the 'then' block
            return interpret(ast_node["then"], variables)
        else 
            -- Interprets the 'otherwise' block
            return interpret(ast_node["otherwise"], variables)
        end
```

### 3.6. Implementando Funções: `Function` e `Call`

Este é o ponto mais complexo, pois envolve a criação de um novo escopo (contexto) para a chamada de função.

*   `Function`: Apenas retorna o próprio nó da AST, que representa a função.
*   `Call`:
    1.  Interpreta o `callee` (a função a ser chamada).
    2.  Cria um **novo escopo** para a execução da função.
    3.  Mapeia os argumentos passados para os parâmetros da função no novo escopo.
    4.  Executa o corpo da função (`value` do nó `Function`) no novo escopo.

```lua
-- Adicione dentro da função interpret

    elseif kind == "Function" then
        -- A função é um valor de primeira classe. Apenas retorna o nó da AST.
        return ast_node
    elseif kind == "Call" then
        -- 1. Interpreta o callee (a função)
        local callee_node = interpret(ast_node["callee"], variables)
        
        -- Verifica se o callee é realmente uma função
        if type(callee_node) ~= "table" or callee_node["kind"] ~= "Function" then
            print("Error: Attempt to call a non-function value.")
            os.exit()
        end

        -- 2. Cria um novo escopo (contexto) para a chamada
        -- Nota: Para simplificar, estamos usando um escopo global modificado.
        -- Em interpretadores mais complexos, o escopo pai seria preservado.
        local new_variables = {}
        
        -- Copia as variáveis do escopo atual para o novo escopo (Closure)
        for name, value in pairs(variables) do
            new_variables[name] = value
        end

        -- 3. Mapeia argumentos para parâmetros
        local parameters = callee_node["parameters"]
        local arguments = ast_node["arguments"]

        if #parameters ~= #arguments then
            print("Error: Function call with incorrect number of arguments.")
            os.exit()
        end

        for i, arg_expression in ipairs(arguments) do
            -- Interpreta o valor do argumento no escopo atual
            local arg_value = interpret(arg_expression, variables)
            -- Atribui o valor ao nome do parâmetro no novo escopo
            local param_name = parameters[i]["text"]
            new_variables[param_name] = arg_value
        end

        -- 4. Executa o corpo da função no novo escopo
        return interpret(callee_node["value"], new_variables)
```

### 3.7. Implementando Tuplas: `Tuple`, `First`, `Second`

Tuplas são coleções ordenadas de dois elementos. Em Lua, podemos representá-las como uma tabela com índices numéricos.

```lua
-- Adicione dentro da função interpret

    elseif kind == "Tuple" then
        -- Cria uma tabela Lua com os dois elementos interpretados
        local first = interpret(ast_node["first"], variables)
        local second = interpret(ast_node["second"], variables)
        return { first, second }
    elseif kind == "First" then
        -- Retorna o primeiro elemento da tupla
        local tuple = interpret(ast_node["value"], variables)
        return tuple[1]
    elseif kind == "Second" then
        -- Retorna o segundo elemento da tupla
        local tuple = interpret(ast_node["value"], variables)
        return tuple[2]
```

## 4. Código Final (`main.lua`)

O código final do seu interpretador (`main.lua`) deve ser uma combinação das seções acima.

```lua
-- main.lua (Código completo em inglês)

json = require "json"

-- The main recursive function to interpret the AST node
function interpret(ast_node, context)
    -- If the node is not a table, it's a literal value (e.g., number, string, boolean)
    if type(ast_node) ~= "table" then
        return ast_node
    end
    
    local kind = ast_node["kind"]
    local variables = context or {}

    -- If no 'kind' is present, it might be a literal or an unhandled structure
    if kind == nil then
        return ast_node
    end

    if kind == "Str" then
        -- Returns the string value
        return ast_node["value"]
    elseif kind == "Int" then
        -- Returns the integer value
        return ast_node["value"]
    elseif kind == "Bool" then
        -- Returns the boolean value
        return ast_node["value"]
    elseif kind == "Print" then
        -- Interprets the value to be printed
        local value = interpret(ast_node["value"], variables)
        -- Prints the value
        print(tostring(value))
        -- Returns the value
        return value
    elseif kind == "Let" then
        -- 1. Interpret the value of the expression
        local value = interpret(ast_node["value"], variables)
        -- 2. Store the value in the context
        variables[ast_node["name"]["text"]] = value
        -- 3. Continue interpreting the next expression
        return interpret(ast_node["next"], variables)
    elseif kind == "Var" then
        -- 1. Retrieve the value from the context
        local value = variables[ast_node["text"]]
        if value == nil then
            error("Error: Variable '" .. ast_node["text"] .. "' not found.")
        end
        -- 2. Return the variable's value
        return value
    elseif kind == "If" then
        -- Interprets the condition
        local condition_result = interpret(ast_node["condition"], variables)
        
        -- If condition is true, interpret 'then' block, otherwise 'otherwise' block
        if condition_result then
            return interpret(ast_node["then"], variables)
        else 
            return interpret(ast_node["otherwise"], variables)
        end
    elseif kind == "Function" then
        -- Returns the AST node itself as a first-class value (closure)
        return ast_node
    elseif kind == "Call" then
        -- 1. Interpret the callee (the function)
        local callee_node = interpret(ast_node["callee"], variables)
        
        if type(callee_node) ~= "table" or callee_node["kind"] ~= "Function" then
            error("Error: Attempt to call a non-function value.")
        end

        -- 2. Create a new scope (context) for the function call
        local new_variables = {}
        
        -- Copy variables from the current scope (basic closure implementation)
        for name, value in pairs(variables) do
            new_variables[name] = value
        end

        -- 3. Map arguments to parameters in the new scope
        local parameters = callee_node["parameters"]
        local arguments = ast_node["arguments"]

        if #parameters ~= #arguments then
            error("Error: Function call with incorrect number of arguments.")
        end

        for i, arg_expression in ipairs(arguments) do
            -- Interpret the argument value in the current scope
            local arg_value = interpret(arg_expression, variables)
            -- Assign the value to the parameter name in the new scope
            local param_name = parameters[i]["text"]
            new_variables[param_name] = arg_value
        end

        -- 4. Execute the function body in the new scope
        return interpret(callee_node["value"], new_variables)
    elseif kind == "Binary" then
        local operator = ast_node["op"]
        local lhs = interpret(ast_node["lhs"], variables)
        local rhs = interpret(ast_node["rhs"], variables)
        
        if operator == "Add" then
            if type(lhs) == "string" or type(rhs) == "string" then
                return lhs .. rhs
            else
                return lhs + rhs
            end
        elseif operator == "Sub" then
            return lhs - rhs
        elseif operator == "Mul" then
            return lhs * rhs
        elseif operator == "Div" then
            if rhs == 0 then
                error("Division by zero error.")
            end
            return math.floor(lhs / rhs)
        elseif operator == "Rem" then
            return lhs % rhs
        elseif operator == "Eq" then
            return lhs == rhs
        elseif operator == "Neq" then
            return lhs ~= rhs
        elseif operator == "Lt" then
            return lhs < rhs
        elseif operator == "Lte" then
            return lhs <= rhs
        elseif operator == "Gt" then
            return lhs > rhs
        elseif operator == "Gte" then
            return lhs >= rhs
        elseif operator == "And" then
            return lhs and rhs
        elseif operator == "Or" then
            return lhs or rhs
        else
            error("Unknown operator: " .. operator)
        end
    elseif kind == "Tuple" then
        -- Creates a Lua table to represent the tuple
        local first = interpret(ast_node["first"], variables)
        local second = interpret(ast_node["second"], variables)
        return { first, second }
    elseif kind == "First" then
        -- Returns the first element of the tuple
        local tuple = interpret(ast_node["value"], variables)
        return tuple[1]
    elseif kind == "Second" then
        -- Returns the second element of the tuple
        local tuple = interpret(ast_node["value"], variables)
        return tuple[2]
    else
        error("Unknown command: " .. kind)
    end
end

-- Main execution block
local input_file = arg[1] or "source.rinha.json"

local file = io.open(input_file, "r")
if file == nil then
    error("Could not read \"" .. input_file .. "\"")
end

local content = file:read "*a"
file:close()

-- Decode the JSON content into a Lua table (the AST)
local ast = json.decode(content)

-- Start interpretation with the main expression and an empty context
interpret(ast["expression"], {})
```

## 5. Testando o Interpretador

Para testar seu interpretador, você usará os arquivos JSON da AST que você forneceu.

**Estrutura de Arquivos:**

```
.
├── main.lua          <-- Seu interpretador
├── json.lua          <-- Módulo JSON
├── fib.uninter.json    <-- Arquivo de teste (AST)
├── sum.uninter.json    <-- Arquivo de teste (AST)
└── ...
```

**Comando de Execução:**

Você executa o interpretador passando o caminho para o arquivo JSON como argumento:

```bash
lua main.lua fib.uninter.json
```

**Exemplos de Teste:**

| Arquivo de Teste | Código Uninter (Original) | Resultado Esperado |
| :--- | :--- | :--- |
| `sum.uninter.json` | `let x = 1 + 2; print(x)` | `3` |
| `print.uninter.json` | `print("Hello" + " World")` | `Hello World` |
| `fib.uninter.json` | Função recursiva de Fibonacci | O 10º número de Fibonacci (55) |

## 6. Próximos Passos e Desafio

Parabéns! Você construiu um interpretador funcional para a linguagem Uninter.

**Desafio de Reuso do Projeto:**

O objetivo final deste projeto é que ele sirva como material de apoio e consulta. Como próximo passo, você pode:

1.  **Publicar no GitHub:** Crie um repositório e publique seu `main.lua`, `json.lua`, este tutorial (`tutorial_interpretador_lua.md`) e os arquivos de teste.
2.  **Implementar o Parser:** O próximo grande passo seria construir a **Análise Léxica** e a **Análise Sintática** para que o interpretador possa ler o código-fonte Uninter (`.uninter`) diretamente, em vez do JSON (`.uninter.json`).).
3.  **Outras Linguagens:** Tente reescrever o interpretador em outra linguagem de programação (Python, JavaScript, C++) para comparar as abordagens.

Este projeto é uma excelente base para aprofundar seus conhecimentos em Compiladores e Interpretadores.
