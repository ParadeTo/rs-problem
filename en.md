I am using [oxc](https://github.com/oxc-project/oxc) to parse and modify JS code.

```js
const b = require('b.js')
```

Take the code above as an example. I want to add a prefix string to the module name(`b.js`), and the prefix is dynamic.

Here is my code as blow, the repository is [here](https://github.com/ParadeTo/rs-problem). The problem is that the `new_name` will be destroyed after function was called, but `Atom::from` declare a lift cycle(`'a`).

So the compiler report the error: `new_name` does not live long enough.

How to resolve this problem?

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
