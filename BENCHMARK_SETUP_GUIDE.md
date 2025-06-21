# Guia de Configuração de Benchmarks no Caliper

Este documento contém as lições aprendidas durante a implementação de benchmarks de desempenho no Hyperledger Caliper para contratos Ethereum.

## Problemas Comuns e Soluções

### 1. Erro "gas is missing"

**Problema**: O Caliper falha com erro `TypeError: Cannot read properties of undefined (reading 'gas')`

**Causa**: Falta de metadados obrigatórios nos arquivos de configuração.

**Solução**:
- **No arquivo JSON do contrato** (`src/ethereum/[ContractName]/[ContractName].json`):
  ```json
  {
    "name": "ContractName",
    "abi": [...],
    "bytecode": "0x...",
    "gas": 4700000
  }
  ```
- **No arquivo de configuração de rede** (`networks/besu/1node-clique/[contract].json`):
  ```json
  {
    "contracts": {
      "ContractName": {
        "path": "./src/ethereum/ContractName/ContractName.json",
        "gas": {
          "functionName1": 200000,
          "functionName2": 100000
        },
        "abi": [...]
      }
    }
  }
  ```

### 2. Alinhamento de Nomes de Funções

**Problema**: Erro persistente de "gas" mesmo com configuração aparentemente correta.

**Causa**: Incompatibilidade entre os nomes das funções no `config.yaml` e nas chaves de gás.

**Solução**: Os `labels` no `config.yaml` devem corresponder exatamente aos nomes das funções no contrato:
```yaml
rounds:
  - label: functionName  # Deve ser igual ao nome da função no contrato
    workload:
      module: benchmarks/scenario/ContractName/function.js
```

### 3. Estrutura de Workload

**Problema**: Sintaxe incorreta de envio de transações.

**Solução**: Use a estrutura comprovadamente funcional:
```javascript
// Em benchmarks/scenario/ContractName/utils/contract-base.js
class ContractBase extends WorkloadModuleBase {
    createConnectorRequest(operation, args) {
        return {
            contract: 'ContractName',
            verb: operation,
            args: args,
            readOnly: (operation === 'queryFunction')
        };
    }
}

// Em benchmarks/scenario/ContractName/function.js
class FunctionWorkload extends ContractBase {
    async submitTransaction() {
        const args = [param1, param2];
        await this.sutAdapter.sendRequests(
            this.createConnectorRequest('functionName', args)
        );
    }
}
```

### 4. Compilação de Contratos

**Problema**: Como gerar ABI e bytecode corretos.

**Solução**: Use o compilador Solidity via Docker:
```bash
# Compilar contrato
docker run --rm -v $(pwd):/workspace ethereum/solc:0.7.6 \
    --abi --bin --optimize --overwrite \
    -o /workspace/src/ethereum/ContractName \
    /workspace/src/ethereum/ContractName/ContractName.sol

# Combinar em JSON
node -e "
const fs = require('fs');
const path = require('path');
const dir = 'src/ethereum/ContractName';
const abi = JSON.parse(fs.readFileSync(path.join(dir, 'ContractName.abi'), 'utf8'));
const bytecode = fs.readFileSync(path.join(dir, 'ContractName.bin'), 'utf8');
const contractJson = {
    name: 'ContractName',
    abi,
    bytecode: '0x' + bytecode,
    gas: 4700000
};
fs.writeFileSync(path.join(dir, 'ContractName.json'), JSON.stringify(contractJson, null, 2));
"
```

### 5. Estado Compartilhado Entre Rodadas

**Problema**: Transações falham porque operam em dados que não existem.

**Solução**: Implemente estado compartilhado:
```javascript
// Em utils/contract-base.js
class ContractState {
    constructor() {
        this.registeredIds = [];
    }
    
    addRegisteredId(id) {
        this.registeredIds.push(id);
    }
    
    getRandomRegisteredId() {
        const randomIndex = Math.floor(Math.random() * this.registeredIds.length);
        return this.registeredIds[randomIndex];
    }
}

const contractState = new ContractState();

class ContractBase extends WorkloadModuleBase {
    async initializeWorkloadModule(...args) {
        await super.initializeWorkloadModule(...args);
        this.contractState = contractState;
    }
}
```

## Estrutura de Arquivos Recomendada

```
benchmarks/scenario/ContractName/
├── config.yaml                 # Configuração do benchmark
├── register.js                 # Workload de registro
├── update.js                   # Workload de atualização
├── query.js                    # Workload de consulta
└── utils/
    └── contract-base.js        # Classe base compartilhada

networks/besu/1node-clique/
└── contract.json              # Configuração de rede

src/ethereum/ContractName/
├── ContractName.sol           # Contrato fonte
├── ContractName.abi           # ABI gerado
├── ContractName.bin           # Bytecode gerado
└── ContractName.json          # Arquivo combinado
```

## Checklist de Configuração

- [ ] Contrato compilado com versão Solidity compatível
- [ ] Arquivo JSON do contrato contém `name`, `abi`, `bytecode`, `gas`
- [ ] Configuração de rede tem seção `gas` para cada função
- [ ] Labels no `config.yaml` correspondem aos nomes das funções
- [ ] Workloads herdam da classe base correta
- [ ] Estado compartilhado implementado se necessário
- [ ] Teste de sanidade executado com benchmark `simple`

## Comandos Úteis

```bash
# Testar benchmark
npx caliper launch manager \
    --caliper-benchconfig benchmarks/scenario/ContractName/config.yaml \
    --caliper-networkconfig networks/besu/1node-clique/contract.json \
    --caliper-workspace .

# Verificar rede
docker compose -f networks/besu/1node-clique/docker-compose.yml ps

# Limpar rede
docker compose -f networks/besu/1node-clique/docker-compose.yml down
```

## Versões Testadas

- Node.js: v18.19.0 (recomendado)
- Caliper: 0.6.0
- Solidity: 0.7.6
- Besu: Latest 