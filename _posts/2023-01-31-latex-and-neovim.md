---
layout: post
title: Neovim and Latex
subtitle: Using Neovim as your text editor for Latex
tags: [lua,neovim,latex]
comments: true
---  

Hi everyone! I have been using Neovim for quite some time now and I have to say, I am in love with it. Over the past few months, I've been exploring the world of Lua and using it to create custom implementations for various workflows in Python.

Recently, I decided to try my hand at using Neovim with Latex as well. Latex is a powerful typesetting system that enables users to create professional-looking documents with ease. I normally use Overleaf to create my Latex documents, but there are times when I don't have an internet connection. 

My past experiences with Latex text editors haven't been too great, so I was on the lookout for a terminal engine that was both efficient and didn't require any extensive setup. That's when I stumbled upon [tectonic](https://tectonic-typesetting.github.io/en-US/)It turned out to be a godsend and works like a charm! All you need to do is run the following command:
```bash
tectonic your-document-name.tex
```

So, I thought it would be cool to automate this process with Neovim. I wrote some Lua functions and was able to achieve this by using the SPACE + C key in normal mode. The scripts that I used are listed below:
```lua 
function replace_char(str, char_to_replace, replacement_char)
   -- INFO: this is for replace the " " for "\ " for bash/zsh paths
  local new_str = ""
  for i = 1, #str do
    local char = string.sub(str, i, i)
    if char == char_to_replace then
      new_str = new_str .. replacement_char
    else
      new_str = new_str .. char
    end
  end
  return new_str
end


function compile_tex()
    -- Compiles the tex file to .pdf

    local command = "tectonic"
    
    local current_file = vim.api.nvim_get_current_buf()
    local current_file = vim.api.nvim_buf_get_name(current_file)
    local current_file = replace_char(current_file," ","\\ ")
    local extension = current_file:match("%.(%w+)$")

    if extension == "tex" then
        local cmd = command .. " " .. current_file
        vim.api.nvim_command("echo'"..cmd.."'")
        local success, output = pcall(vim.api.nvim_call_function, "system", {cmd})
        if success then
            vim.api.nvim_command("echo '"..output.."'")
        else
            vim.api.nvim_command("echo 'Error: "..output.."'")
        end
    else
        vim.api.nvim_command("echo 'Not .text file in buffer'")
    end
end

vim.api.nvim_set_keymap("n", "<leader>c", ":lua compile_tex()<CR>", {noremap = true})
```
This script checks the file that is currently in the buffer and then compiles it into a PDF document. It's that simple! I hope this post helps those of you who are interested in trying to implement Neovim in your text-editing workflow.
