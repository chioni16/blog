---
title: "Linux Foundation Mentorship: Solang"
date: 2023-11-15T13:49:58+05:30
---

In June 2023, I was selected for Linux Foundation mentorship to contribute to the [Solang compiler](https://github.com/hyperledger/solang). Solang is a Solidty compiler that targets Solana and Polkadot. It is written in Rust and uses LLVM as the backend. The mentorship focused on enriching the language server's functionalities, which would make for a better development experience.

### Working
The language server operates in conjunction with the compiler, using the Abstract Syntax Tree (AST) it generates. The language server traverses the AST and extracts information like location, type, scope information, among other things, of various symbols present in the source code. This is repeated everytime there are changes in the source code. The extracted information is later used to serve user requests like `goto-definition`, `rename` etc.

Code for language server can be found [here](https://github.com/hyperledger/solang/blob/main/src/bin/languageserver/mod.rs).

### Features
Here are the features offered by the language server. Descriptions of these features are accompanied by videos demonstrating their use, available under the `Video` tab.

#### Go to Definition
{{< tabs tabTotal="2" >}}
{{% tab tabName="Description" %}}
Returns the definition of a variable or a type defined in user code.
{{% /tab %}}
{{< tab tabName="Video" >}}
{{< youtube xAbUsloLnhk >}}
{{< /tab >}}
{{< /tabs >}}
#### Go to Declaration
{{< tabs tabTotal="2" >}}
{{% tab tabName="Description" %}}
Returns a list of methods that the given method overrides.
{{% /tab %}}
{{< tab tabName="Video" >}}
{{< youtube YKwK1c-443I >}}
{{< /tab >}}
{{< /tabs >}}
#### Go to Implementation
{{< tabs tabTotal="2" >}}
{{% tab tabName="Description" %}}
Returns a list of methods defined for the given contract.
{{% /tab %}}
{{< tab tabName="Video" >}}
{{< youtube 313WH44ta7s >}}
{{< /tab >}}
{{< /tabs >}}
#### Go to Type Definition
{{< tabs tabTotal="2" >}}
{{% tab tabName="Description" %}}
Returns the type of the given symbol (variable, struct field, contract method etc).
{{% /tab %}}
{{< tab tabName="Video" >}}
{{< youtube bxc8FRpiMiI >}}
{{< /tab >}}
{{< /tabs >}}
#### Go to References
{{< tabs tabTotal="2" >}}
{{% tab tabName="Description" %}}
Returns a list of uses of the given symbol.
{{% /tab %}}
{{< tab tabName="Video" >}}
{{< youtube BWvbi9qz1mU >}}
{{< /tab >}}
{{< /tabs >}}
#### Rename
{{< tabs tabTotal="2" >}}
{{% tab tabName="Description" %}}
Renames the given symbol.
{{% /tab %}}
{{< tab tabName="Video" >}}
{{< youtube 1FSD_HwMb5I >}}
{{< /tab >}}
{{< /tabs >}}
#### Code Completion
{{< tabs tabTotal="2" >}}
{{% tab tabName="Description" %}}
Offers suggestions to the user based on contextual information and lexical scoping. Pressing `.` yields the properties (such as struct fields, enum variants, contract methods etc.) associated with the preceding symbol.
{{% /tab %}}
{{< tab tabName="Video" >}}
{{< youtube p_z3s6XEJzk >}}
{{< /tab >}}
{{< /tabs >}}
#### Format Code
{{< tabs tabTotal="2" >}}
{{% tab tabName="Description" %}}
Formats user code using [forge-fmt](https://lib.rs/crates/forge-fmt) crate.
{{% /tab %}}
{{< tab tabName="Video" >}}
{{< youtube sWPwPoQ7hEg >}}
{{< /tab >}}
{{< /tabs >}}
#### Hovers
{{< tabs tabTotal="2" >}}
{{% tab tabName="Description" %}}
Displays information about the symbol under the cursor.
{{% /tab %}}
{{< tab tabName="Video" >}}
{{< youtube laTmPwmbMFA  >}}
{{< /tab >}}
{{< /tabs >}}

### Improvements
Here are some ideas for enhancements:
- The language server relies on compiler's parser to extract information from the source code. However, the parser tends to prematurely halt in the presence of errors and may not parse the entire program. This hinders language server's ability to offer information or suggestions to the user during code editing. We need to make the parser more resilient to errors.
- Incorporate missing source code locations of various symbols in the parse tree.

### Takeaways
Contributing to open source projects can be a very fulfilling and enjoyable experience. It helps one foster connections with like-minded individuals and work on interesting projects together, which would otherwise be simply not possible.