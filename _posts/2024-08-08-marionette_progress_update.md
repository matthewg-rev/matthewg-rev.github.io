## Marionette Progress Update 2

Finally had the chance to work on Marionette again today. I have been working on the Lua lexer and have made progress. Currently multi-line comments are now supported inside the lexer. With that the only improvement to syntax highlighting in terms of functionality would be to implement function call highlighting, however that is not a huge priority at the moment. Another future improvement down the line that I would like to make is implementing optimizations to the lexer so that it does not lex the entire content of the widget every time the content changes. This would be a huge performance improvement.

I believe there may be optimizations to be found in the lexing function for these multi-line strings and comments however the lexing library logos does not support backtracking in regex matching and this was the first solution I could come up with:
```rust
#[derive(Logos, Debug, PartialEq)]
enum LuaToken {
    // ...

    #[regex(r"--", lua_comment, priority = 1)]
    Comment,

    // ...

    #[regex(r#""([^"\\]|\\["\\bnfrt]|u[a-fA-F0-9]{4})*""#)]
    #[regex(r"'[^']*'")]
    #[regex(r"\[=*\[", lua_string)]
    String,

    // ...
}

#[derive(Logos, Debug, PartialEq)]
enum LuaMultiLineString {
    #[token("[", priority = 10)]
    Open,
    #[token("]", priority = 10)]
    Close,
    #[token("=", priority = 10)]
    Equals,
    #[regex(".")]
    Content
}

fn lua_string(lex: &mut logos::Lexer<LuaToken>) -> Result<(), ()> {
    let remainder = lex.remainder();
    let mut chars = remainder.chars().peekable();
    let mut eq_count = 0;

    // get currently matched data
    let mut data = lex.slice().to_string();
    eq_count = data.len() - 2; // remove opening brackets from length

    println!("remainder: {:?}", remainder);

    // define the closing sequence
    let closing_sequence = format!("]{}]", "=".repeat(eq_count));

    // push characters into content until closing sequence is found
    let mut content = String::new();
    while let Some(c) = chars.next() {
        content.push(c);
        if content.ends_with(&closing_sequence) {
            break;
        }
    }

    // check closing sequence
    if !content.ends_with(&closing_sequence) {
        return Err(());
    }

    // bump the lexer
    let comment_len = remainder.len() - chars.collect::<String>().len();
    lex.bump(comment_len);

    return Ok(());
}

// ...

#[derive(Logos, Debug, PartialEq)]
enum LuaComment {
    #[token("[", priority = 10)]
    Open,
    #[token("]", priority = 10)]
    Close,
    #[token("=", priority = 10)]
    Equals,
    #[regex(".")]
    Content
}

fn lua_comment(lex: &mut logos::Lexer<LuaToken>) -> Result<(), ()> {
    let remainder = lex.remainder();
    let mut chars = remainder.chars().peekable();
    let mut multi_line = false;

    if chars.peek() == Some(&'[') {
        chars.next();
        multi_line = true;
    }

    println!("remainder: {:?} | multi_line: {:?}", remainder, multi_line);

    if multi_line {
        // count the number of equals signs
        let mut eq_count = 0;
        while let Some('=') = chars.peek() {
            chars.next();
            eq_count += 1;
        }

        // make sure the enter bracket is there
        if chars.next() != Some('[') {
            return Err(());
        }

        // define the closing sequence
        let closing_sequence = format!("]{}]", "=".repeat(eq_count));

        // push characters into content until closing sequence is found
        let mut content = String::new();
        while let Some(c) = chars.next() {
            content.push(c);
            if content.ends_with(&closing_sequence) {
                break;
            }
        }

        // check closing sequence
        if !content.ends_with(&closing_sequence) {
            return Err(());
        }

        // bump the lexer
        let comment_len = remainder.len() - chars.collect::<String>().len();
        lex.bump(comment_len);

        return Ok(());
    }

    // Single-line comment
    while let Some(c) = chars.peek() {
        if c == &'\n' {
            break;
        }
        chars.next();
    }

    let comment_len = remainder.len() - chars.collect::<String>().len();
    lex.bump(comment_len);

    return Ok(())
}
```

To wrap up this post, I have included an image of the Marionette text widget with syntax highlighting enabled as well as a list of features that the text-widget plans on supporting or already has supported:
- [ ] Lua syntax highlighting.
    - [x] Basic syntax highlighting.
    - [x] Multi-line comment and string support.
    - [ ] Bytecode highlighting.
- [ ] Python syntax highlighting.
    - [x] Basic syntax highlighting.
    - [x] Multi-line comment and string support.
    - [ ] Bytecode highlighting.
- [ ] Function call highlighting.
- [ ] Optimizations to the lexer.
    - [ ] Only lex the content that has changed.
    - [ ] Port the widget to rust to decrease lexing latency.

![Marionette text widget syntax highlighting example](https://github.com/matthewg-rev/matthewg-rev.github.io/images/2024-08-08-marionette_progress_update/marionette_app_TycBOan2E1.png)