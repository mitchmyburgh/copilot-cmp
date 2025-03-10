# copilot-cmp

This repository transforms https://github.com/zbirenbaum/copilot.lua into a cmp source.

Copilot suggestions will automatically be loaded into your cmp menu as snippets and display their full contents when a copilot suggestion is hovered.

![copilot-cmp](https://user-images.githubusercontent.com/32016110/173933674-9ad85a5a-5ad7-41cd-9fcc-f5a698cc88ae.png)


## Setup

If you already have copilot.lua installed, you can install this plugin with packer as you would any other with the following code:

### Install

```lua
use {
  "zbirenbaum/copilot-cmp",
  after = { "copilot.lua" },
  config = function ()
    require("copilot_cmp").setup()
  end
}
```

If you do not have copilot.lua installed, go to https://github.com/zbirenbaum/copilot.lua and follow the instructions there before installing this one

It is recommended to disable copilot.lua's suggestion and panel modules, as they can interfere with completions properly appearing in copilot-cmp. To do so, simply place the following in your copilot.lua config:
```lua
require("copilot").setup({
  suggestion = { enabled = false },
  panel = { enabled = false },
})
```

### Configuration

##### Default Options:
These are the default options for copilot-cmp which can be configured via the setup function:
```lua
{
  formatters = {
    label = require("copilot_cmp.format").format_label_text,
    insert_text = require("copilot_cmp.format").format_insert_text,
    preview = require("copilot_cmp.format").deindent,
  },
}
```

##### clear_after_cursor
(10-09-22): Due to changes in cmp, this option will cause the whole line to be deleted, so I have removed it until it is clear whether this behavior is a result of intended or buggy behavior. Fortunately, the issue this option was implemented to fix seems to no longer be a problem you use 'replace' for the confirmation behavior.

cmp config example:
```lua
cmp.setup({
  mapping = {
    ["<CR>"] = cmp.mapping.confirm({
      -- this is the important line
      behavior = cmp.ConfirmBehavior.Replace,
      select = false,
    }),
  }
})

```

##### formatters
The `label` field corresponds to the function returning the label of the entry in nvim-cmp, `insert_text` corresponds to the actual text that is inserted, and `preview` corresponds to the text shown in the documentation window when hovering the completion.

There is an experimental method for attempting to remove extraneous characters such as extra ending parenthesis that appears to work fairly well. If you wish to use it, simply place the following in your setup function:
```lua
{
  formatters = {
    insert_text = require("copilot_cmp.format").remove_existing
  },
}


```
Additionally, each field can be overriden by a user function that takes (item, ctx) as parameters and should return a string representing what you wish to supply cmp. This is an advanced feature to provide maximum configurability for how the source will format the raw output from Copilot. These functions can be extremely complex, and this is feature only recommended for advanced users, so taking a look at how the default functions work inside of `format.lua` will likely be necessary if you wish to override the behavior.
##### Source Definition

To link cmp with this source, simply go into your cmp configuration file and include `{ name = "copilot" }` under your sources

Here is an example of what it should look like:

```lua
cmp.setup {
  ...
  sources = {
    -- Copilot Source
    { name = "copilot", group_index = 2 },
    -- Other Sources
    { name = "nvim_lsp", group_index = 2 },
    { name = "path", group_index = 2 },
    { name = "luasnip", group_index = 2 },
  },
  ...
}
```

##### Highlighting & Icon

Copilot's cmp source now has a builtin highlight group `CmpItemKindCopilot`. To add an icon to copilot for lspkind, simply add copilot to your lspkind symbol map.

```lua
-- lspkind.lua
local lspkind = require("lspkind")
lspkind.init({
  symbol_map = {
    Copilot = "",
  },
}

vim.api.nvim_set_hl(0, "CmpItemKindCopilot", {fg ="#6CC644"})
```

Alternatively, you can add Copilot to the lspkind `symbol_map` within the cmp format function.

```lua
-- cmp.lua
cmp.setup {
  ...
  formatting = {
    format = lspkind.cmp_format({
      mode = "symbol",
      max_width = 50,
      symbol_map = { Copilot = "" }
    })
  }
  ...
}
```

If you do not use lspkind, simply add the custom icon however you normally handle `kind` formatting and it will integrate as if it was any other normal lsp completion kind.

##### Tab Completion Configuration (Highly Recommended)
Unlike other completion sources, copilot can use other lines above or below an empty line to provide a completion. This can cause problematic for individuals that select menu entries with `<TAB>`. This behavior is configurable via cmp's config and the following code will make it so that the menu still appears normally, but tab will fallback to indenting unless a non-whitespace character has actually been typed.

```lua
local has_words_before = function()
  if vim.api.nvim_buf_get_option(0, "buftype") == "prompt" then return false end
  local line, col = unpack(vim.api.nvim_win_get_cursor(0))
  return col ~= 0 and vim.api.nvim_buf_get_text(0, line-1, 0, line-1, col, {})[1]:match("^%s*$") == nil
end
cmp.setup({
  mapping = {
    ["<Tab>"] = vim.schedule_wrap(function(fallback)
      if cmp.visible() and has_words_before() then
        cmp.select_next_item({ behavior = cmp.SelectBehavior.Select })
      else
        fallback()
      end
    end),
  },
})
```

##### Comparators

One custom comparitor for sorting cmp entries is provided: `prioritize`. The `prioritize` comparitor causes copilot entries to appear higher in the cmp menu. It is recommended keeping priority weight at 2, or placing the `exact` comparitor above copilot so that better lsp matches are not stuck below poor copilot matches.

Example:

```lua
cmp.setup {
  ...
  sorting = {
    priority_weight = 2,
    comparators = {
      require("copilot_cmp.comparators").prioritize,

      -- Below is the default comparitor list and order for nvim-cmp
      cmp.config.compare.offset,
      -- cmp.config.compare.scopes, --this is commented in nvim-cmp too
      cmp.config.compare.exact,
      cmp.config.compare.score,
      cmp.config.compare.recently_used,
      cmp.config.compare.locality,
      cmp.config.compare.kind,
      cmp.config.compare.sort_text,
      cmp.config.compare.length,
      cmp.config.compare.order,
    },
  },
  ...
}
```
