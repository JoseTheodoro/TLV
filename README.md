TLV
Padrão de arquitetura de encoding para organizar a mensagem inteira
Organizar os dados em 3 blocos para que o computador saiba exatamente o que está lendo.
Os 3 blocos são:
- Type: Qual o tipo do dado: Nome, Idade ou Email?
- Length: Quantos bytes esse dados ocupa?
- Value: Os dados reais
Geralmente é definido um tamanho fixo para o Type e para o Length: Type: 1 Byte, Length: 2 bytes e Value é o len dos dados em si Bytes, por exemplo "Ana" usaria 3 Bytes. Totalizando em uma mensagem de 6 bytes. TLV tem uma limitação tradicional: Vamos usar o nome "Ana" como exemplo: O len é 3. Que em bytes seria [00000000][00000011]. O primeiro byte foi totalmente desperdiçado só com zeros. É exatamente nessa limitação que entra o algoritmo LEB 128
Mesmo assim o TLV é padrão na indústria, é mais performático ter os dados organizados, previsíveis mesmo que gaste um pouco a mais de bytes. É o trade-off clássico, gastamos um pouco mais em bytes em troca de flexibilidade, compatibilidade e velocidade de parsing.
Não recomendado para sistemas de altíssima performance, estrutura de dados extremamente rígida, consumo de banda e memória incrivelmente restrito.


Funcionamento
Vamos especificar um mini protocolo binário de exemplo, mas antes, e apenas para reforçar LTV não é Fixed Position, como muitos protocolos binários usam um Header Fixed para trafegar informações simples e pequenas.

Reforçando os 3 blocos do TLV:

Campo
Posição
Tamanho(bytes)
Type
Descrição
Type
0
1
uint8
Identificador único do tipo da informação
Length
1
2
uint16
Tamanho exato do Value em bytes
Value
3
variável
[]byte
Os dados reais da mensagem



Especificação do protocolo binário para trafegar dados de pessoas, acabei de criar da minha cabeça. Apenas didático.

Type
Nome do campo
Tipo do dado
exemplo
0x01 (1)
Nome
String 
Winter
0x02 (2)
Idade
Int 
33
0x03 (3)
Email
String 
winter@tinnybots.com




Imaginamos que o protocolo tem uma especificação para identificar o objeto que está chegando.

Type
Entitidade
exemplo
0x0A (10)
Pessoa
Pessoa{Nome, Idade, Email}



Com o protocolo binário para trafegar uma pessoa na rede Pessoa struct{} vamos encodar.

Vamos montar em 3 partes:
Primeiro vamos encodar o Nome
Segundo vamos encodar a Idade
Terceiro vamos encodar o email

Depois criaremos um único slice de bytes para compor e trafegar pela rede.

Encodar o Nome:
Nome de exemplo: Winter
O protocolo pede que 1 byte seja reservado para especificar o tipo, como é o nome precisa ser 1(decimal)
typeNome := make([]byte, 0x01)
TypeNome[0] = 1


Agora o Length: Pede que seja 2 bytes // para essa parte eu poderia usar o binary.BigEndian.PutUint16 ao invés do append
LenNome := make([]byte, 2)
LenNome = append(LenNome, byte(len(“winter”)))

Agora o Value:
ValueNome :make([]byte len(“winter”))

packetName := make([]byte, 3+len(“winter”))
packetName[0] TypeNome)
packetName[1]= append(packetName, LenNome)
packetName = append(packetName, byte(ValueNome)...)

packetName ficou com 3+6= 9 bytes.

Encodar Idade:
Idade de exemplo: 33 uint8


O protocolo pede que 1 byte seja reservado para especificar o tipo, como é a idade precisa ser 2(decimal)
TypeIdade := make([]byte, 0x01)
TypeIdade[0] = 2

Para o Length: 2 bytes
LenIdade := make([]byte, 2)
LenIdade = append(LenIdade, 1)

Agora o Value:
ValueIdade := make([]byte, 1)

packetIdade := make([]byte, 3+1)
packetIdade = append(packetIdade, typeIdade)
packetIdade = append(packetIdade, LenIdade)
packetIdade = append(packetIdade, byte(ValueIdade)...)

packetIdade ficou com 4 bytes 3fixos pelo LTV e 1 para idade uint8


Encodar Email:

Email de exemplo: “winter@abc.com” // len=14
TypeEmail := make([]byte, 0x03) // protocolo pede que para email seja o decimal 3
TypeEmail[0] = 3

Length:
LenEmail := make([]byte, 2) // protocolo pede que sejam dois bytes
Lenemail = append(LenEmail, len(“winter@abc.com”))

Value:
ValueEmail := make([]byte, len(“winter@abc.com”))

packetEmail := make([]byte, 3+len(“winter@abc.com”))
packetEmail = append(packetEmail, typeEmail)
packetEmail = append(packetEmail, LenEmail)
packetEmail = append(packetEmail, byte(“winter@abc.com”)...)

packetEmail ficou com 3+14 = 17bytes
Total 30bytes

Encodamos Nome, Idade e Email usando o LTV.
Agora precisamos encodar tudo em um só pacote para transporte. Pacote Pessoa, podemos dizer.
Quem também é um LTV. Poderia dizer que para ser Pessoa, poderia ser do type 0x0A 

typePessoa := make([]byte, 0x01)
typePessoa[0] = 10


LenPessoa := make([]byte, 2)
LenPessoa = append(LenPessoa, len(packetNome)+len(packetIdade)+len(PacketEmail)) // 30bytes

ValuePessoa := make([]byte, len(packetNome)+len(packetIdade)+len(PacketEmail))
ValuePessoa = append(ValuePessoa, packetNome)
ValuePessoa = append(ValuePessoa, packetIdade)
ValuePessoa = append(ValuePessoa, packetEmail)


Ao todo: 33 bytes (30 bytes Nome, Idade e Email + 3 bytes para o type Pessoa)

Bora codar!!!

Conclusão
É um pseudocódigo para ilustrar a implementação de como funciona a ordenação do TLV para trafegar dados binários. 
Dividimos Nome em 3 blocos. 
O primeiro informando o que é. Pelo nosso protocolo que criamos para trafegar dados de pessoas, é decimal 1
O segundo foi informado o tamanho do Nome, reservado em 2 bytes.
O terceiro foi guardar os bytes do valor do nome.
Empacotamos tudo em um pacote de nome, idade, email e Pessoa como especificava o protocolo que criamos Exemplo:
Quando esses dados trafegarem pela rede e chegarem ao decode, o decode saberá exatamente onde começa e termina cada Nome, Idade, Email


Nome Ordenado por TLV
1 byte
2 byte
3:6 bytes
1
6
winter
9 bytes



Idade Ordenado por TLV
1 byte
2 byte
3:4 bytes
2
1
33
3 bytes



Email Ordenado por TLV
1 byte
2 byte
3:14 bytes
3
14
winter@abc.com
17 bytes



Pessoa Ordenado por TLV
1 byte
2 byte
3: bytes
10
30
Winter 33 winter@abc.com
33 bytes

