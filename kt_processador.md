# fluxo processador ler pedidos/vendas:

## Setando processador a ser executado
--------------------------

1. A classe Main vai chamar a classe *ProcessamentoFactory*

2. Na classe *ProcessamentoFactory* é criado um objeto Processamento.

	- esse Processamento possui um metodo executarProcesso que recebe um array de parâmetros
	
	- dentro do metodo *instanciar* ele pega o numero do processador pelos argumentos. 

	- com o numero do processador ele retorna o processador correspondente a aquele numero do mapaConfiguracoes que tem na classe.


3. Ainda na classe main é chamado o metodo *executarProcesso* do Procesamento retornado do passo anterior.

	- O metodo executar processo não vai estar no escopo da classe Processamento retornada, mas sim dentro da classe *ProcessamentoBase* que é herdada pelo processamento em questão:
	
		- incializa contexto: serviços de grupoEmpresa, rede...
		-  chama o metodo executar() que é abstract, então o Processamento específico dos passos anteriores é que vai ter esse executar
		
	- Dentro de *Processador35* vai ter o metodo *executar()*
	
		- É feito umas montagens de monitoramento que relevo por agora
		
		- é criado um bean do ProcessadorMarketplace que é analogo ao ProcessadorFato do processador01
		
		- processadorMarketplace.iniciarProcessador passando array de id empresas e array de id marketplaces
		
			* obs: esses ids de empresa e rede foram capturados pelos argumentos passados
			
			- para cada marketplace id é chamado o metodo iniciarProcessador passsando o id marketplace e a lista de id de empresas
			
				- vai ter um switch que vai cair no mkt especifico do id passado.
				- vai chamar processadorFulano.processar(passando parametros empresa ID).
		
		
## processar()
--------------------------

primeiramente o que é chamado pra executar é o metodo processar() que não possui na classe *ProcessadorMarketplace*. Ele fica no *ProcessadorBaseService* que é herado.
	
* obs: O ProcessadorMarketplace herda de ProcessadorMarketplaceServiceBeanIO que herda de ProcessadorBaseService
	
1. no metodo *processar()* é chamado o metodo *preparaListaArquivos()* passando o path para o diretorio com arquivos a serem processados.
    - é feita uma analise se o nome do path para cara arquivo esta correto e ai so retorna os paths validados
         - para cada arquivo analisado vai ser verificado com regex se na nomeclatura do arquivo há o ID de uma das empresasID do array passado.
		- retorna um novo array de arquivos so com os que foram encontrada empresaId no nome.
		
	- proximo passo é aplicar filtro de descrição header: é passado a lista de arquivos do passo anterior para o metodo que faz a aplicação de filtro descricao do header
		
	- com a lista de arquivos ja resultante dos filtros, agora ele vai pegar o Rede (mktplace) pelo id: ele ta indo no banco então pra pegar só o obj rede
        - obs: talvez fosse interessante colocar em um REDIS pra buscar mais rapido.
		
	- caso a Rede tenha `rede.getLoop() == true;`  Significa q ela precisa agrupar arquivos
		
	- move os arquivos para o diretorio Temp, usando o PathsProcessador
		
	-  ordena os arquivos
		
	- é chamado o metodo *processarArquivos* passando a lista de arquivos. O metodo *processarArquivos* não fica na *ProcessadorBaseService*. Ele fica na *ProcessadorMarketplaceServiceBeanIO*
		- é feito um loop pra cada arquivo:
			- atualiza o status do arquivo na base para EM PROCESSAMENTO e passando o nome do arquivo e data
			- chama logErroProcessadorService para removerLogPorNomeArquivo
			- monta um BeanReader leitor pro arquivo
			- vai no banco buscar o objeto Empresa (aqui da tbm pra otimizar, pois ta indo buscar a cada arquivo!!)
			- vai ser feita a logica de mapeamento do arquivo e VOs
				- é montado o mapaVOs chamando o metodo defineMapeamentos que fica no NetshoesService
				- pra cada linha processada do arquivo, é chamado o metodo processarResultados passando o mapaVOs, arquivo e linha


### Mapeamento de records em VOs
--------------------------
obs: O processo abaixo é disparado atráves do método *processarResultados*, linha 56,na classe *ProcessadorMarketplaceServiceBeanIO* dentro do loop do array de arquivos. Então o processo abaixo é executado por cada arquivo:

1. Vai ser criado o BeanReader que é a variavel *leitor*. Esse beanReader é montado pelo método *getLeitorArquivo(arquivo)*:
    - dentro do método, vai ser configurado os registros para dentro de StreamFactory do BeanIO.
    - o metodo que monta esses registros é o *defineRegistros* que recebe o arquivo como parâmetro:
        - é feito varias addRecord para configurar o BeanIO na hora do mapeamento das records em objetos:

        ```java
            return new StreamBuilder(READER_NAME).format(DELIMITED)
                    .parser(new DelimitedParserBuilder(DELIMITER))
                    .ignoreUnidentifiedRecords()
                    .addRecord(TransacaoVendaNetshoesVO.class)
                    .addRecord(TransacaoDevolucaoNetshoesVO.class)
                    .addRecord(TransacaoTrocaNetshoesVO.class)
                    .addRecord(TransacaoAjusteEDevolucaoETrocaNetshoesVO.class)
                    .addRecord(TransacaoCicloPagamentoNetshoesVO.class)
                    .addTypeHandler(Double.class, NetshoesDoubleTypeHandler.class.newInstance())
                    .addTypeHandler(Date.class, NetshoesDateTypeHandler.class.newInstance());
        ```

    - esses VOs possuem umas configurações que são usadas peo BeanIO


2. Para o método *defineMapeamentos* são passados 2 parâmetros porém não são usados.
    obs: Assim é feito a chamada para o método: 

    `MapeamentoProcessador mapeamentoProcessador = defineMapeamentos(arquivo, mapaVOs);`

3. Método *defineMapeamento*:
    - esse método retorna um *MapeamentoProcessador*
    - Vale lembrar que esse método fica implementado dentro de *ProcessadorMarketPlaceNethoesService*
    - é montado um List de classes VOs:
        ```java
        classes.add(TransacaoVendaNetshoesVO.class);
        classes.add(TransacaoDevolucaoNetshoesVO.class);
        classes.add(TransacaoTrocaNetshoesVO.class);
        classes.add(TransacaoAjusteEDevolucaoETrocaNetshoesVO.class);
        classes.add(TransacaoCicloPagamentoNetshoesVO.class);
        ```
    - e essas classes são mapeadas para processadoresServices, correspondente para cada uma dessas classes. E então é feito o mapeamento:
        ```java
        mapService.put(TransacaoVendaNetshoesVO.class, processadorMarketPlaceNethoesTransacaoVendaService);
        mapService.put(TransacaoDevolucaoNetshoesVO.class, processadorMarketPlaceNethoesTransacaoDevolucaoService);
        mapService.put(TransacaoTrocaNetshoesVO.class, processadorMarketPlaceNethoesTransacaoTrocaService);
        mapService.put(TransacaoAjusteEDevolucaoETrocaNetshoesVO.class, processadorMarketPlaceNethoesAjusteEDevolucaoETrocaService);
        ```
    - o método termina retornando um objeto de *MapeamentoProcessador* que é montado passando no construtor tanto o **mapService** e o **classes** acima.

        obs: O mapa ira indicar qual service sera chamado no momento do processamento do VO. A lista de classes ira indicar as classes dos VOs que serao usadas durante o processamento na sequencia.

4. O retorno de MapeamentoProcessador do metodo anterior é colocado na variavel *mapeamentoProcessador*;

5. É feito a lógica para ler cada linha do arquivo:
    - dentro do mapaVOs vai ser adicionado o nome do arquivo e o numero da linha processada;
    - É chamado o método *processarResultados* passando **record**, **mapeamentoProcessador**, **arquivo**, **mapaVOs**:
        - é feito um for por todas as classes que foram adicionadas no mapeamentoProcessador:
            - adiciona a classe no mapaVOs
            - invoca metodo *setNomeArquivo* do processador do VO especifico
            - invoca metodo processar do processador do VO especifico (EXPLICAREI DEPOIS)
    
6. é chamadado o método posProcessamentoArquivo que seta informações de IR



### Executando os Invocadores de Processadores Service VO
--------------------------
Aqui vai ser explicado o que é feito nos invocadores de setnar Nome do arquivo e processar da etapa 5 explicada na seção anterior.

1. como descrito anteriormente, o metodo que chama esses invocadores é o metodo 
```java
processarResultados(Object record, MapeamentoProcessador mapeamentoProcessador, File arquivo, Map<Class<?>, Object> mapaVOs)
``` 
da classe *ProcessadorMarketplaceServiceBeanIO*

2. Nele há dois metodos de invocar:

```java
invocaMetodoSetNomeArquivo(arquivo, mapaVOs, clazz);
invocarMetodoProcessar(clazz, mapaVOs, mapeamentoProcessador.getMapaServices());
```

3. invocaMetodoSetNomeArquivo:
    - esse método vai buscar o método *setNomeArquivo* e vai fazer o .invoke desse método.
    
    obs: ainda não entendi de quem ele puxa esse metodo setNomeArquivo, pois os VOs do Marketplace Netshoes não tem esse método. Acredito que seja do AbstractVO

4. invocarMetodoProcessar:
    - esse método recebe como parâmetros:
        - clazz: a classe VO
        - mapaVOs: mapa que é usado como aux no processo
        - mapaServices: map de classVO -> processadorVO que fica no *MapeamentoProcessador*
    
    - primeiro ele pega a Class que representa o serviceVO através do mapServices

    - depois ele busca o método que está na Class:
        ```java
        for (Method method : serviceClass.getMethods()) {
			if (method.getName().equals(NOME_METODO_PROCESSAR)) {
				return method;
			}
		}
        ```
    - se for null, ele joga exception

    - uma vez que possui o método processar que foi capturado como dito acima, ele vai chamar o .invoke desse método passando argumentos

    -  esses argumentos são passados para o metodo processar através de uma lógica bem complexa, onde ele monta os parametros por um array...

        obs: acho que não precisamos disso.
    
    - Os argumentos passados para o metodo *processar()* de todos os services são:
        - o VO em questão
        - empresa
        - rede
        - TransacaoCicloPagamentoVO
    
    - Dentro do processar de qualquer VO vem a lógica de montar a FatoMarketplace:
            
            obs: Irei falar da logica do tipo VENDA:
        - monta a fato padrão passando o VO, empresa, rede e cicloPagamentoVO;
        - seta o tipo de pedido (VENDA, ...);
        - seta alguns valores especificos como valor pago, data pagamento...
        - salva a fato chamando o service:
            ```java 
            fatoMarketPlaceService.salvarFatoMarketPlace(fatoMarketPlace)
            ```
        OBS: Nesse ponto de salvar na fato é o que seria feito pelo WRITER do Spring Batch!



### Salvando dados na base
-------------------
A última seção terminou com o processo de chamar o fatoMarketplaceService.salvarFatoMarketplace... essa seção vai destrinchar o que acontece com essa chamada de método:

1. primeiro é feita uma consulta se a fatoMarketplace passada por parâmetro para o método existe. 

2. Caso exista, vai ser chamado o método *salvarFatoMarketPlaceSQLNativo*

    - É montada um array de campos da tabela
    - Cria-se o sql de INSERT pegando de `fatoMarketplace.getEmpresa().getId()` para concatenar e formar o nome da tabela na base. EX: *FATO_MARKETPLACE_022
    - seta os parametros do fatoMarketplace no sql INSERT
    - chama o query.executeUpdate()
