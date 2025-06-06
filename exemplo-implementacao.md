# Exemplo de Implementação: Servidor de Memória com Grafo de Conhecimento

Este exemplo demonstra como implementar um servidor de memória baseado em grafo de conhecimento usando JavaScript/Node.js.

## Estrutura do Projeto

```
memory-server/
├── server.js
├── models/
│   ├── Entity.js
│   ├── Relation.js
│   └── KnowledgeGraph.js
├── routes/
│   ├── entities.js
│   ├── relations.js
│   └── queries.js
└── package.json
```

## Implementação das Classes Base

### Entity.js
```javascript
class Entity {
    constructor(id, type, attributes = {}) {
        this.id = id;
        this.type = type;
        this.attributes = attributes;
        this.createdAt = new Date();
        this.updatedAt = new Date();
    }

    updateAttribute(key, value) {
        this.attributes[key] = value;
        this.updatedAt = new Date();
    }

    getAttribute(key) {
        return this.attributes[key];
    }

    toJSON() {
        return {
            id: this.id,
            type: this.type,
            attributes: this.attributes,
            createdAt: this.createdAt,
            updatedAt: this.updatedAt
        };
    }
}

module.exports = Entity;
```

### Relation.js
```javascript
class Relation {
    constructor(id, fromEntityId, toEntityId, type, weight = 1.0, attributes = {}) {
        this.id = id;
        this.fromEntityId = fromEntityId;
        this.toEntityId = toEntityId;
        this.type = type;
        this.weight = weight;
        this.attributes = attributes;
        this.createdAt = new Date();
        this.updatedAt = new Date();
    }

    updateWeight(newWeight) {
        this.weight = newWeight;
        this.updatedAt = new Date();
    }

    updateAttribute(key, value) {
        this.attributes[key] = value;
        this.updatedAt = new Date();
    }

    toJSON() {
        return {
            id: this.id,
            fromEntityId: this.fromEntityId,
            toEntityId: this.toEntityId,
            type: this.type,
            weight: this.weight,
            attributes: this.attributes,
            createdAt: this.createdAt,
            updatedAt: this.updatedAt
        };
    }
}

module.exports = Relation;
```

### KnowledgeGraph.js
```javascript
const Entity = require('./Entity');
const Relation = require('./Relation');

class KnowledgeGraph {
    constructor() {
        this.entities = new Map();
        this.relations = new Map();
        this.entityTypes = new Set();
        this.relationTypes = new Set();
    }

    // Métodos para Entidades
    addEntity(id, type, attributes = {}) {
        if (this.entities.has(id)) {
            throw new Error(`Entity with id ${id} already exists`);
        }
        
        const entity = new Entity(id, type, attributes);
        this.entities.set(id, entity);
        this.entityTypes.add(type);
        return entity;
    }

    getEntity(id) {
        return this.entities.get(id);
    }

    updateEntity(id, attributes) {
        const entity = this.entities.get(id);
        if (!entity) {
            throw new Error(`Entity with id ${id} not found`);
        }
        
        Object.keys(attributes).forEach(key => {
            entity.updateAttribute(key, attributes[key]);
        });
        
        return entity;
    }

    deleteEntity(id) {
        // Remove todas as relações que envolvem esta entidade
        const relationsToDelete = [];
        this.relations.forEach((relation, relationId) => {
            if (relation.fromEntityId === id || relation.toEntityId === id) {
                relationsToDelete.push(relationId);
            }
        });
        
        relationsToDelete.forEach(relationId => {
            this.relations.delete(relationId);
        });
        
        return this.entities.delete(id);
    }

    // Métodos para Relações
    addRelation(id, fromEntityId, toEntityId, type, weight = 1.0, attributes = {}) {
        if (!this.entities.has(fromEntityId) || !this.entities.has(toEntityId)) {
            throw new Error('Both entities must exist before creating a relation');
        }
        
        if (this.relations.has(id)) {
            throw new Error(`Relation with id ${id} already exists`);
        }
        
        const relation = new Relation(id, fromEntityId, toEntityId, type, weight, attributes);
        this.relations.set(id, relation);
        this.relationTypes.add(type);
        return relation;
    }

    getRelation(id) {
        return this.relations.get(id);
    }

    updateRelation(id, updates) {
        const relation = this.relations.get(id);
        if (!relation) {
            throw new Error(`Relation with id ${id} not found`);
        }
        
        if (updates.weight !== undefined) {
            relation.updateWeight(updates.weight);
        }
        
        if (updates.attributes) {
            Object.keys(updates.attributes).forEach(key => {
                relation.updateAttribute(key, updates.attributes[key]);
            });
        }
        
        return relation;
    }

    deleteRelation(id) {
        return this.relations.delete(id);
    }

    // Métodos de Consulta
    getEntitiesByType(type) {
        const result = [];
        this.entities.forEach(entity => {
            if (entity.type === type) {
                result.push(entity);
            }
        });
        return result;
    }

    getRelatedEntities(entityId, relationType = null, direction = 'both') {
        const related = [];
        
        this.relations.forEach(relation => {
            if (relationType && relation.type !== relationType) {
                return;
            }
            
            let targetEntityId = null;
            
            if ((direction === 'outgoing' || direction === 'both') && relation.fromEntityId === entityId) {
                targetEntityId = relation.toEntityId;
            } else if ((direction === 'incoming' || direction === 'both') && relation.toEntityId === entityId) {
                targetEntityId = relation.fromEntityId;
            }
            
            if (targetEntityId && this.entities.has(targetEntityId)) {
                related.push({
                    entity: this.entities.get(targetEntityId),
                    relation: relation
                });
            }
        });
        
        return related;
    }

    searchEntities(query) {
        const results = [];
        const queryLower = query.toLowerCase();
        
        this.entities.forEach(entity => {
            // Busca no tipo da entidade
            if (entity.type.toLowerCase().includes(queryLower)) {
                results.push({ entity, score: 1.0, matchType: 'type' });
                return;
            }
            
            // Busca nos atributos
            Object.values(entity.attributes).forEach(value => {
                if (typeof value === 'string' && value.toLowerCase().includes(queryLower)) {
                    results.push({ entity, score: 0.8, matchType: 'attribute' });
                }
            });
        });
        
        return results.sort((a, b) => b.score - a.score);
    }

    getContextForEntity(entityId, depth = 2) {
        const context = {
            entity: this.getEntity(entityId),
            related: [],
            paths: []
        };
        
        if (!context.entity) {
            return null;
        }
        
        // Busca entidades relacionadas diretamente
        const directlyRelated = this.getRelatedEntities(entityId);
        context.related = directlyRelated;
        
        // Busca caminhos mais profundos se depth > 1
        if (depth > 1) {
            directlyRelated.forEach(({ entity: relatedEntity }) => {
                const secondLevelRelated = this.getRelatedEntities(relatedEntity.id);
                secondLevelRelated.forEach(({ entity: secondLevelEntity, relation }) => {
                    if (secondLevelEntity.id !== entityId) {
                        context.paths.push({
                            from: entityId,
                            through: relatedEntity.id,
                            to: secondLevelEntity.id,
                            relation: relation
                        });
                    }
                });
            });
        }
        
        return context;
    }

    // Métodos de Exportação/Importação
    export() {
        return {
            entities: Array.from(this.entities.values()).map(e => e.toJSON()),
            relations: Array.from(this.relations.values()).map(r => r.toJSON()),
            metadata: {
                entityTypes: Array.from(this.entityTypes),
                relationTypes: Array.from(this.relationTypes),
                exportedAt: new Date()
            }
        };
    }

    import(data) {
        // Limpa o grafo atual
        this.entities.clear();
        this.relations.clear();
        this.entityTypes.clear();
        this.relationTypes.clear();
        
        // Importa entidades
        data.entities.forEach(entityData => {
            const entity = new Entity(entityData.id, entityData.type, entityData.attributes);
            entity.createdAt = new Date(entityData.createdAt);
            entity.updatedAt = new Date(entityData.updatedAt);
            this.entities.set(entity.id, entity);
            this.entityTypes.add(entity.type);
        });
        
        // Importa relações
        data.relations.forEach(relationData => {
            const relation = new Relation(
                relationData.id,
                relationData.fromEntityId,
                relationData.toEntityId,
                relationData.type,
                relationData.weight,
                relationData.attributes
            );
            relation.createdAt = new Date(relationData.createdAt);
            relation.updatedAt = new Date(relationData.updatedAt);
            this.relations.set(relation.id, relation);
            this.relationTypes.add(relation.type);
        });
    }
}

module.exports = KnowledgeGraph;
```

## Servidor Express

### server.js
```javascript
const express = require('express');
const cors = require('cors');
const KnowledgeGraph = require('./models/KnowledgeGraph');

const app = express();
const port = process.env.PORT || 3000;

// Middleware
app.use(cors());
app.use(express.json());

// Instância global do grafo de conhecimento
const knowledgeGraph = new KnowledgeGraph();

// Rotas para Entidades
app.post('/api/entities', (req, res) => {
    try {
        const { id, type, attributes } = req.body;
        const entity = knowledgeGraph.addEntity(id, type, attributes);
        res.status(201).json(entity.toJSON());
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

app.get('/api/entities/:id', (req, res) => {
    const entity = knowledgeGraph.getEntity(req.params.id);
    if (!entity) {
        return res.status(404).json({ error: 'Entity not found' });
    }
    res.json(entity.toJSON());
});

app.put('/api/entities/:id', (req, res) => {
    try {
        const entity = knowledgeGraph.updateEntity(req.params.id, req.body);
        res.json(entity.toJSON());
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

app.delete('/api/entities/:id', (req, res) => {
    const deleted = knowledgeGraph.deleteEntity(req.params.id);
    if (!deleted) {
        return res.status(404).json({ error: 'Entity not found' });
    }
    res.status(204).send();
});

// Rotas para Relações
app.post('/api/relations', (req, res) => {
    try {
        const { id, fromEntityId, toEntityId, type, weight, attributes } = req.body;
        const relation = knowledgeGraph.addRelation(id, fromEntityId, toEntityId, type, weight, attributes);
        res.status(201).json(relation.toJSON());
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

app.get('/api/relations/:id', (req, res) => {
    const relation = knowledgeGraph.getRelation(req.params.id);
    if (!relation) {
        return res.status(404).json({ error: 'Relation not found' });
    }
    res.json(relation.toJSON());
});

// Rotas de Consulta
app.get('/api/entities', (req, res) => {
    const { type, search } = req.query;
    
    if (search) {
        const results = knowledgeGraph.searchEntities(search);
        return res.json(results);
    }
    
    if (type) {
        const entities = knowledgeGraph.getEntitiesByType(type);
        return res.json(entities.map(e => e.toJSON()));
    }
    
    // Retorna todas as entidades se não houver filtros
    const allEntities = Array.from(knowledgeGraph.entities.values());
    res.json(allEntities.map(e => e.toJSON()));
});

app.get('/api/entities/:id/related', (req, res) => {
    const { type, direction } = req.query;
    const related = knowledgeGraph.getRelatedEntities(req.params.id, type, direction);
    res.json(related.map(r => ({
        entity: r.entity.toJSON(),
        relation: r.relation.toJSON()
    })));
});

app.get('/api/entities/:id/context', (req, res) => {
    const depth = parseInt(req.query.depth) || 2;
    const context = knowledgeGraph.getContextForEntity(req.params.id, depth);
    
    if (!context) {
        return res.status(404).json({ error: 'Entity not found' });
    }
    
    res.json({
        entity: context.entity.toJSON(),
        related: context.related.map(r => ({
            entity: r.entity.toJSON(),
            relation: r.relation.toJSON()
        })),
        paths: context.paths
    });
});

// Rota para exportar/importar dados
app.get('/api/export', (req, res) => {
    const data = knowledgeGraph.export();
    res.json(data);
});

app.post('/api/import', (req, res) => {
    try {
        knowledgeGraph.import(req.body);
        res.json({ message: 'Data imported successfully' });
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

app.listen(port, () => {
    console.log(`Memory server running on http://localhost:${port}`);
});
```

## Package.json
```json
{
  "name": "memory-server",
  "version": "1.0.0",
  "description": "Knowledge Graph Memory Server",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

## Como Usar

### 1. Instalação
```bash
npm install
npm start
```

### 2. Exemplos de Uso

#### Criar uma Entidade
```bash
curl -X POST http://localhost:3000/api/entities \
  -H "Content-Type: application/json" \
  -d '{
    "id": "user-1",
    "type": "Person",
    "attributes": {
      "name": "João Silva",
      "profession": "Developer",
      "preferences": ["JavaScript", "AI", "Music"]
    }
  }'
```

#### Criar uma Relação
```bash
curl -X POST http://localhost:3000/api/relations \
  -H "Content-Type: application/json" \
  -d '{
    "id": "rel-1",
    "fromEntityId": "user-1",
    "toEntityId": "project-1",
    "type": "WORKS_ON",
    "weight": 0.8,
    "attributes": {
      "role": "Lead Developer",
      "since": "2024-01-01"
    }
  }'
```

#### Buscar Contexto
```bash
curl http://localhost:3000/api/entities/user-1/context?depth=2
```

## Integração com IA Generativa

Para integrar com Claude ou outra IA, você pode usar este prompt:

```
Você tem acesso a um servidor de memória baseado em grafo de conhecimento em http://localhost:3000/api.

Antes de responder, sempre:
1. Busque o contexto do usuário: GET /api/entities/{user-id}/context
2. Se relevante, busque entidades relacionadas: GET /api/entities/{entity-id}/related
3. Após a interação, atualize as informações: POST/PUT nas rotas apropriadas

Use as informações do grafo para personalizar suas respostas e manter continuidade entre conversas.
```

Esta implementação fornece uma base sólida para sistemas de memória persistente em aplicações de IA generativa.