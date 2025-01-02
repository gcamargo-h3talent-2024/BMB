### Contexto e Aplicação das Promessas no Código

O código do processing-scripts utiliza promessas para gerenciar operações assíncronas, como consultas a bancos de dados e processamento de dados. As promessas são uma maneira eficiente de lidar com operações que podem levar algum tempo para serem concluídas, permitindo que o código continue a ser executado enquanto se aguarda a conclusão dessas operações.

### Criação de Dados de Requisição de Compras

A função `createPurchaseRequisitionData` é responsável por criar os dados das requisições de compras. Ela utiliza a função `Sys.Helpers.Promise.Create` para criar uma nova promessa. Dentro dessa promessa, uma consulta é configurada para buscar requisições de compras. A consulta é executada e os resultados são processados em um loop. Se todos os registros forem processados com sucesso, a promessa é resolvida (`resolve`). Se houver uma discrepância no número de registros processados, a promessa é rejeitada (`reject`).

#### Código:
```javascript
function createPurchaseRequisitionData() {
  return Sys.Helpers.Promise.Create(function (resolve, reject) {
    var nbOfRecords = 0;
    var transport;
    purchaseRequisitionDataArray.push(initpurchaseRequisitionDataArray());
    var queryM = Process.CreateQueryAsProcessAdmin();
    queryM.Reset();
    queryM.SetOptionEx("Limit=5000");
    buildPRQueryFilter(queryM);
    queryM.SetSpecificTable("CDNAME#Purchase requisition");
    queryM.SetAttributesList("*");
    queryM.SetSearchInArchive(true);
    queryM.SetSortOrder("RequisitionNumber__ ASC");
    transport = queryM.MoveFirst();

    if (transport) {
      nbOfRecords = queryM.GetRecordCount();
      if (nbOfRecords == 0) {
        Data.SetValue("Mensagem__", "Não foram encontradas RCs com os dados informados");
      }
      transport = queryM.MoveNext();
      while (transport) {
        purchaseRequisitionData.push(initpurchaseRequisitionData());
        var vars = transport.GetUninheritedVars();
        if (vars.GetValue_String("RequisitionNumber__", 0)) {
          rc = vars.GetValue_String("RequisitionNumber__", 0);
        }
        purchaseRequisitionData[loops].TipoDocumento = "Compras";
        purchaseRequisitionData[loops].RC = vars.GetValue_String("RequisitionNumber__", 0);
        transport = queryM.MoveNext();
        loops++;
      }
      if (loops == nbOfRecords) {
        resolve("Youhou");
      } else {
        reject("Diferenca loops:" + loops + " nbOfRecords: " + nbOfRecords);
      }
    } else {
      if (transport.GetLastError()) {
        reject(transport.GetLastErrorMessage());
      }
    }
  });
}
```

### Consulta de Gestão de Contratos

A função `queryGestaoDeContratos` consulta informações de gestão de contratos. Ela também utiliza `Sys.Helpers.Promise.Create` para criar uma nova promessa. O filtro da consulta é configurado com base no contrato ou `ruid`. A consulta é executada para buscar informações de contratos. Se a consulta for bem-sucedida, a promessa é resolvida com os dados do contrato. Se não encontrar o contrato ou ocorrer um erro, a promessa é resolvida com uma mensagem de erro ou rejeitada.

#### Código:
```javascript
function queryGestaoDeContratos(purchaseRequisitionData, tipocompra, ruid) {
  return Sys.Helpers.Promise.Create(function (resolve, reject) {
    var tablec = "CDNAME#Gestao de Contratos";
    var filterc;
    if (purchaseRequisitionData.Contrato) {
      filterc = "(Numero_Documento__=" + purchaseRequisitionData.Contrato + ")";
    } else {
      filterc = "(RuidEx=CD#GGBW2F6F9A3." + ruid + ")";
    }
    var QueryC = Process.CreateQueryAsProcessAdmin();
    QueryC.Reset();
    QueryC.SetSpecificTable(tablec);
    QueryC.SetFilter(filterc);
    QueryC.SetAttributesList("*");
    QueryC.SetSearchInArchive(true);
    if (QueryC.MoveFirst()) {
      var transportc = QueryC.MoveNext();
      if (transportc) {
        var varsc = transportc.GetUninheritedVars();
        purchaseRequisitionData.ContratoStatus = varsc.GetValue_String("Status__", 0);
        purchaseRequisitionData.ContratoOwner = varsc.GetValue_String("OwnerID__", 0);
        var c = purchaseRequisitionData.ContratoOwner.split(",");
        var owner = c[0].substring(3, c[0].length);
        purchaseRequisitionData.ContratoOwner = owner;
        resolve("Query GC successful");
      } else {
        resolve("Nao achou Contrato " + filterc);
      }
    } else {
      reject("Move First GC error");
    }
  });
}
```

### Consulta de Pedidos de Compra (PO)

A função `queryPO` consulta informações de pedidos de compra (PO). Ela utiliza `Sys.Helpers.Promise.Create` para criar uma nova promessa. O filtro da consulta é configurado com base no número da requisição. A consulta é executada para buscar informações de pedidos de compra. Se a consulta for bem-sucedida, a promessa é resolvida com os dados do pedido de compra. Se não encontrar o pedido ou ocorrer um erro, a promessa é rejeitada.

#### Código:
```javascript
function queryPO(purchaseRequisitionData) {
  return Sys.Helpers.Promise.Create(function (resolve, reject) {
    var tablePO = "CDNAME#Purchase order";
    var filterPO = "(RequisitionNumber__=" + purchaseRequisitionData.RC + ")";
    var QueryPO = Process.CreateQueryAsProcessAdmin();
    QueryPO.Reset();
    QueryPO.SetSpecificTable(tablePO);
    QueryPO.SetFilter(filterPO);
    QueryPO.SetAttributesList("*");
    QueryPO.SetSearchInArchive(true);
    if (QueryPO.MoveFirst()) {
      var transportPO = QueryPO.MoveNext();
      if (transportPO) {
        var varspo = transportPO.GetUninheritedVars();
        purchaseRequisitionData.PO = varspo.GetValue_String("OrderNumber__", 0);
        purchaseRequisitionData.DataPO = varspo.GetValue_String("Data_Pedido__", 0);
        purchaseRequisitionData.Buyer = varspo.GetValue_String("BuyerName__", 0);
        resolve("Query PO successful");
      } else {
        reject("queryPO error");
      }
    } else {
      reject("Move First queryPO error");
    }
  });
}
```

### Consulta de Notas Fiscais (NF)

A função `queryNF` consulta informações de notas fiscais (NF). Ela utiliza `Sys.Helpers.Promise.Create` para criar uma nova promessa. O filtro da consulta é configurado com base no número do pedido de compra (PO). A consulta é executada para buscar informações de notas fiscais. Se a consulta for bem-sucedida, a promessa é resolvida com os dados da nota fiscal. Se não encontrar a nota ou ocorrer um erro, a promessa é resolvida com uma mensagem de erro.

#### Código:
```javascript
function queryNF(purchaseRequisitionData) {
  return Sys.Helpers.Promise.Create(function (resolve, reject) {
    var tablenfpo = "CDNAME#Notas_Fiscais";
    var filternfpo = "(&(Pedido_de_Compras__=" + purchaseRequisitionData.PO + ") (Validation__!=FIRST)(Validation__!=DESTINATION)(State!=300)(State!=400)(Deleted=0))";
    var QueryNFPO = Process.CreateQueryAsProcessAdmin();
    QueryNFPO.Reset();
    QueryNFPO.SetSpecificTable(tablenfpo);
    QueryNFPO.SetFilter(filternfpo);
    QueryNFPO.SetAttributesList("*");
    QueryNFPO.SetSearchInArchive(true);
    QueryNFPO.SetOptionEx("Limit=2000");
    QueryNFPO.MoveFirst();
    var transportnf = QueryNFPO.MoveNext();
    if (transportnf) {
      var varsnf = transportnf.GetUninheritedVars();
      purchaseRequisitionData.DataNF = varsnf.GetValue_String("Data_Fatura__", 0);
      purchaseRequisitionData.DataPG = varsnf.GetValue_String("Dt_Basica__", 0);
      purchaseRequisitionData.PassoNF = varsnf.GetValue_String("Validation__", 0);
      purchaseRequisitionData.MontanteNF = varsnf.GetValue_String("Valor_Total_Nota__", 0);
      resolve("Query NF successful");
    } else {
      resolve("Query NF nao encontrou / filternfpo:" + filternfpo);
    }
  });
}
```

### Conclusão

Esses trechos de código mostram como as promessas são usadas para gerenciar operações assíncronas, como consultas a bancos de dados, e como os dados são processados e armazenados em arrays para uso posterior na geração de relatórios. As promessas ajudam a manter o código organizado e a lidar com possíveis erros de maneira estruturada.
