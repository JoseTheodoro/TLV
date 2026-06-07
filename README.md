# TLV (Type-Length-Value)

**Padrão de arquitetura de encoding binário** para organizar a mensagem inteira. Consiste em organizar os dados em 3 blocos para que o computador saiba exatamente o que está lendo.

Os 3 blocos são:

* **Type:** Qual o tipo do dado (Nome, Idade ou Email?)
* **Length:** Quantos bytes esse dado ocupa?
* **Value:** Os dados reais.

Geralmente é definido um tamanho fixo para o Type e para o Length. Sendo: `Type: 1 Byte`, `Length: 2 bytes` e o `Value` é o `len()` dos dados em si. Por exemplo, a string `"Ana"` usaria 3 Bytes. Totalizando uma mensagem de 6 bytes.

> **A Limitação Tradicional do TLV**
> Vamos usar o nome "Ana" como exemplo: O tamanho (length) é 3. Que em bytes de 16 bits (2 bytes) seria `[00000000] [00000011]`. O primeiro byte foi totalmente desperdiçado só com zeros. É exatamente nessa limitação de espaço que entra o algoritmo [**LEB128**](https://github.com/JoseTheodoro/LEB128).

Mesmo assim, o TLV é padrão na indústria. É mais performático ter os dados organizados e previsíveis, mesmo que gaste um pouco mais de bytes. É o *trade-off* clássico: gastamos um pouco mais em bytes em troca de flexibilidade, compatibilidade e velocidade de *parsing*.

Não recomendado para sistemas de altíssima performance com estrutura de dados extremamente rígida, onde o consumo de banda e memória são incrivelmente restritos.

---

## Funcionamento

Vamos especificar um mini protocolo binário de exemplo. Apenas para reforçar: **TLV não é Fixed Position**, como muitos protocolos binários usam um Header fixo para trafegar informações simples e pequenas.

### Reforçando os 3 blocos do TLV:

| Campo | Posição | Tamanho (bytes) | Tipo | Descrição |
| --- | --- | --- | --- | --- |
| **Type** | 0 | 1 | `uint8` | Identificador único do tipo da informação |
| **Length** | 1 | 2 | `uint16` | Tamanho exato do Value em bytes |
| **Value** | 3 | variável | `[]byte` | Os dados reais da mensagem |

---

## Especificação do Protocolo Binário

*Criado para trafegar dados de pessoas (exemplo didático).*

### Dicionário de Tipos de Dados

| Type | Nome do campo | Tipo do dado | Exemplo |
| --- | --- | --- | --- |
| `0x01` (1) | Nome | String | Winter |
| `0x02` (2) | Idade | Int | 33 |
| `0x03` (3) | Email | String | winter@tinnybots.com |

### Identificação de Entidade (Objeto)

Imaginamos que o protocolo tem uma especificação para identificar o objeto raiz que está chegando.

| Type | Entidade | Exemplo |
| --- | --- | --- |
| `0x0A` (10) | Pessoa | `Pessoa{Nome, Idade, Email}` |

---

## Implementação (Pseudocódigo)

Com o protocolo binário para trafegar uma pessoa na rede (`Pessoa struct{}`), vamos encodar. Vamos montar em 3 partes:

1. Primeiro vamos encodar o Nome.
2. Segundo vamos encodar a Idade.
3. Terceiro vamos encodar o Email.

Depois criaremos um único slice de bytes para compor e trafegar pela rede.

### 1. Encodar o Nome

Nome de exemplo: `"Winter"`

```go
// O protocolo pede que 1 byte seja reservado para especificar o tipo.
// Como é o nome, precisa ser 1 (decimal)
typeNome := make([]byte, 0x01)
TypeNome[0] = 1

// Agora o Length: Pede que seja 2 bytes 
// (Para essa parte eu poderia usar o binary.BigEndian.PutUint16 ao invés do append)
LenNome := make([]byte, 2)
LenNome = append(LenNome, byte(len("winter")))

// Agora o Value:
ValueNome := make([]byte, len("winter"))

packetName := make([]byte, 3+len("winter"))
packetName[0] = TypeNome
packetName[1] = append(packetName, LenNome)
packetName = append(packetName, byte(ValueNome)...)

// packetName ficou com 3 + 6 = 9 bytes.

```

### 2. Encodar Idade

Idade de exemplo: `33` (`uint8`)

```go
// O protocolo pede que 1 byte seja reservado para especificar o tipo.
// Como é a idade, precisa ser 2 (decimal)
TypeIdade := make([]byte, 0x01)
TypeIdade[0] = 2

// Para o Length: 2 bytes
LenIdade := make([]byte, 2)
LenIdade = append(LenIdade, 1)

// Agora o Value:
ValueIdade := make([]byte, 1)

packetIdade := make([]byte, 3+1)
packetIdade = append(packetIdade, typeIdade)
packetIdade = append(packetIdade, LenIdade)
packetIdade = append(packetIdade, byte(ValueIdade)...)

// packetIdade ficou com 4 bytes: 3 fixos pelo TLV e 1 para idade uint8.

```

### 3. Encodar Email

Email de exemplo: `"winter@abc.com"` (`len=14`)

```go
// Protocolo pede que para email seja o decimal 3
TypeEmail := make([]byte, 0x03) 
TypeEmail[0] = 3

// Length: protocolo pede que sejam dois bytes
LenEmail := make([]byte, 2) 
Lenemail = append(LenEmail, len("winter@abc.com"))

// Value:
ValueEmail := make([]byte, len("winter@abc.com"))

packetEmail := make([]byte, 3+len("winter@abc.com"))
packetEmail = append(packetEmail, typeEmail)
packetEmail = append(packetEmail, LenEmail)
packetEmail = append(packetEmail, byte("winter@abc.com")...)

// packetEmail ficou com 3 + 14 = 17 bytes.
// Total até aqui: 30 bytes

```

### 4. O Pacote Raiz (A Entidade Pessoa)

Encodamos Nome, Idade e Email usando o TLV. Agora precisamos encodar tudo em um só pacote para transporte. Podemos chamar de "Pacote Pessoa", que também é um TLV. Pelo protocolo, para ser Pessoa, o Type é `0x0A`.

```go
typePessoa := make([]byte, 0x01)
typePessoa[0] = 10

LenPessoa := make([]byte, 2)
// Somando o tamanho interno dos filhos: 30 bytes
LenPessoa = append(LenPessoa, len(packetNome)+len(packetIdade)+len(PacketEmail)) 

ValuePessoa := make([]byte, len(packetNome)+len(packetIdade)+len(PacketEmail))
ValuePessoa = append(ValuePessoa, packetNome)
ValuePessoa = append(ValuePessoa, packetIdade)
ValuePessoa = append(ValuePessoa, packetEmail)

// Ao todo: 33 bytes (30 bytes Nome, Idade e Email + 3 bytes para o header type Pessoa)

```

**Bora codar!!!**

---

## Conclusão

É um pseudocódigo para ilustrar a implementação de como funciona a ordenação do TLV para trafegar dados binários.
Dividimos as informações em 3 blocos. 

    1. O primeiro informando o que é. 
    2. O segundo informando o tamanho, reservado em 2 bytes. 
    3. O terceiro guardando os bytes do valor real.

Empacotamos tudo em um pacote composto `Pessoa` contendo `Nome`, `Idade` e `Email` como especificava o protocolo que criamos.

Quando esses dados trafegarem pela rede e chegarem ao `decode`, ele saberá exatamente onde começa e termina cada campo. Veja a estrutura final na memória:

### Nome Ordenado por TLV

**Total: 9 bytes**

| 1 byte (Type) | 2 bytes (Length) | 6 bytes (Value) |
| --- | --- | --- |
| 1 | 6 | winter |

### Idade Ordenada por TLV

**Total: 4 bytes** 

| 1 byte (Type) | 2 bytes (Length) | 1 byte (Value) |
| --- | --- | --- |
| 2 | 1 | 33 |

### Email Ordenado por TLV

**Total: 17 bytes**

| 1 byte (Type) | 2 bytes (Length) | 14 bytes (Value) |
| --- | --- | --- |
| 3 | 14 | winter@abc.com |

### Pessoa Ordenada por TLV (Pacote Completo)

**Total: 33 bytes**

| 1 byte (Type) | 2 bytes (Length) | 30 bytes (Value: Filhos) |
| --- | --- | --- |
| 10 | 30 | `[Bloco Nome] [Bloco Idade] [Bloco Email]` |