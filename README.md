# NAVE. Catering — Menu Ordering System

Sistema de encomenda de menus para eventos NAVE Academy.

## Ficheiros

- `index.html` — Interface de encomenda (cliente)
- `menu-data.json` — Dados dos menus (editar aqui para actualizar menus)

## Setup inicial

### 1. Configurar token Fibery

Em `index.html`, linha ~20, substituir:
```js
fiberyToken: 'FIBERY_API_TOKEN',
```
pelo token gerado em navepadel.fibery.io → Settings → API.

### 2. Publicar no GitHub Pages

1. Criar repositório no GitHub (ex: `nave-menu`)
2. Fazer upload dos dois ficheiros
3. Ir a Settings → Pages → Branch: main → Save
4. URL fica: `https://[username].github.io/nave-menu/`

### 3. Domínio personalizado (opcional)

Para usar `menu.navepadel.pt`:
1. Em Settings → Pages → Custom domain → escrever `menu.navepadel.pt`
2. No DNS do domínio, adicionar registo CNAME:
   - Nome: `menu`
   - Valor: `[username].github.io`

## Actualizar menus

Editar `menu-data.json` directamente no GitHub (Edit file → Commit).
As alterações ficam live imediatamente.

## Estrutura do menu-data.json

```json
{
  "menus": [{
    "id": "mb1",
    "code": "Menu MB 1",
    "pricePerPerson": 30,
    "minGuests": 20,
    "courses": [{
      "id": "mb1-entradas",
      "selectionRule": "choose_exactly | choose_min | choose_max | choose_range",
      "minSelections": 1,
      "maxSelections": 2,
      "dishes": [...]
    }]
  }]
}
```

## Fibery — campos criados em Orders

| Campo | Tipo |
|-------|------|
| Client Name | Text |
| Client Email | Text |
| Client Phone | Text |
| Company | Text |
| Event Date | Date |
| Guest Count | Number |
| Notes | Text |
| Menu Name | Text |
| Price Per Person | Number |
| Total Estimate | Number |
| Selections | Text (JSON) |
| Status | Select |
