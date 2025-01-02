Claro! Aqui estão alguns trechos de código do arquivo `Processing_scripts(com-promisse).pdf` que exemplificam o uso de promessas (Promises) para gerenciar operações assíncronas:

### 1. Criação de Dados de Requisição de Compras
```javascript
function createPurchaseRequisitionData() {
  return Sys.Helpers.Promise.Create(function (resolve, reject) {
    // Inicialização de variáveis e configuração da consulta
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
        // População dos dados de requisição de compras
        purchaseRequisitionData[loops].TipoDocumento = "Compras";
        purchaseRequisitionData[loops].RC = vars.GetValue_String("RequisitionNumber__", 0);
        // ... (continuação da população dos dados)
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

### 2. Consulta de Gestão de Contratos
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

### 3. Consulta de Pedidos de Compra (PO)
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

### 4. Consulta de Notas Fiscais (NF)
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
