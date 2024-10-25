我正在使用 [oxc](https://github.com/oxc-project/oxc) 来解析并修改 JS 代码。

```js
const b = require('b.js')
```

如上所示，我想给模块名(`b.js`)加上一个前缀，这个前缀是动态的。

下面是我的代码，代码仓库见[这里](https://github.com/ParadeTo/rs-problem)。现在的问题是 `new_name` 会在函数执行完后销毁，但是 `Atom::from` 声明了一个生命周期(`'a`)，所以编译器报错：`new_name` does not live long enough.

怎么解决这个问题呀？跪求大佬指点。

```rs
#![allow(clippy::print_stdout)]
use itertools::Itertools;
use oxc_allocator::Allocator;
use oxc_ast::ast::*;
use oxc_codegen::{CodeGenerator, CodegenOptions};
use oxc_parser::Parser;
use oxc_semantic::SemanticBuilder;
use oxc_span::SourceType;
use oxc_traverse::{traverse_mut, Traverse, TraverseCtx};
use std::ops::DerefMut;
use std::{env, path::Path, sync::Arc};

struct MyTransform {
    prefix: String,
}

impl<'a> Traverse<'a> for MyTransform {
    fn enter_call_expression(&mut self, node: &mut CallExpression<'a>, ctx: &mut TraverseCtx<'a>) {
        if node.is_require_call() {
            let argument: &mut Argument<'a> = &mut node.arguments.deref_mut()[0];
            match argument {
                Argument::StringLiteral(string_literal) => {
                    let old_name = string_literal.value.as_str();
                    let new_name = format!("{}{}", self.prefix, old_name);

                    // !!!!!! `new_name` does not live long enough
                    string_literal.value = Atom::from(new_name.as_str());
                }
                _ => {}
            }
        }
    }
}

fn main() -> std::io::Result<()> {
    let name = env::args().nth(1).unwrap_or_else(|| "test.js".to_string());
    let path = Path::new(&name);
    let source_text = Arc::new(std::fs::read_to_string(path)?);
    let source_type = SourceType::from_path(path).unwrap();

    // Memory arena where Semantic and Parser allocate objects
    let allocator = Allocator::default();

    // 1 Parse the source text into an AST
    let parser_ret = Parser::new(&allocator, &source_text, source_type).parse();
    if !parser_ret.errors.is_empty() {
        let error_message: String = parser_ret
            .errors
            .into_iter()
            .map(|error| format!("{:?}", error.with_source_code(Arc::clone(&source_text))))
            .join("\n");
        println!("Parsing failed:\n\n{error_message}",);
        return Ok(());
    }

    let mut program = parser_ret.program;

    // 2 Semantic Analyze
    let semantic = SemanticBuilder::new(&source_text)
        .build_module_record(path, &program)
        // Enable additional syntax checks not performed by the parser
        .with_check_syntax_error(true)
        .build(&program);

    if !semantic.errors.is_empty() {
        let error_message: String = semantic
            .errors
            .into_iter()
            .map(|error| format!("{:?}", error.with_source_code(Arc::clone(&source_text))))
            .join("\n");
        println!("Semantic analysis failed:\n\n{error_message}",);
    }
    let (symbols, scopes) = semantic.semantic.into_symbol_table_and_scope_tree();

    // 3 Transform
    let t = &mut MyTransform {
        prefix: "ayou".to_string(),
    };
    traverse_mut(t, &allocator, &mut program, symbols, scopes);

    // 4 Generate Code
    let new_code = CodeGenerator::new()
        .with_options(CodegenOptions {
            ..CodegenOptions::default()
        })
        .build(&program)
        .code;

    println!("{}", new_code);

    Ok(())
}
```
