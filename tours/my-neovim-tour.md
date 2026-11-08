全く触ってないために操作を忘れているため、
懇切丁寧に操作を教えるページ

#vim

### Plugin mannager
`:Lazy`

### LSP
`:Mason`

### ファイルを探す
`,ff`

### ターミナルを開く
`,tc`, `,tt`など

## 閲覧
### [fold](https://vim-jp.org/vimdoc-ja/fold.html)
- `zo`: fold open

### messages
```lua
local bufopts = { noremap = true, silent = true, buffer = bufnr }
vim.keymap.set('n', ']]', '<Cmd>Lspsaga diagnostic_jump_next<CR>', bufopts)
vim.keymap.set('n', ']e', '<Cmd>Lspsaga show_cursor_diagnostics<CR>', bufopts)

-- Definitions
vim.keymap.set('n', ']d', '<Cmd>Lspsaga peek_definition<CR>', bufopts)
vim.keymap.set('n', ']D', vim.lsp.buf.definition, bufopts)
vim.keymap.set('n', ']<C-d>', vim.lsp.buf.definition, bufopts)

-- Implementations
vim.keymap.set('n', ']i', '<Cmd>Lspsaga finder imp<CR>', bufopts)
vim.keymap.set('n', ']I', vim.lsp.buf.implementation, bufopts)

vim.keymap.set('n', ']f', '<Cmd>Lspsaga finder<CR>', bufopts)
```
## 移動
motion lead key: `m`
- `m<Space>`: fuzzymotion
- `mj`: jumpcursor