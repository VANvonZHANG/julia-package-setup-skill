# Julia Package Structure Template

## Project.toml

```toml
name = "YourPackageName"
uuid = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"  # generate with uuid4()
version = "0.1.0"

[deps]
# Runtime dependencies go here
# Example:
# StaticArrays = "90137ffa-7385-5640-81b9-e52037218182"

[compat]
julia = "1.10"
# Add compat bounds for each dependency
# Example:
# StaticArrays = "1.9"

[extras]
# Test-only dependencies
Test = "8dfed614-e22c-5e08-85e1-65c5234f0b40"
Aqua = "4c88cf16-eb10-579e-8560-4a9242c79595"

[targets]
test = ["Test", "Aqua"]
```

**Generate UUID**: In Julia REPL, run `using UUIDs; uuid4()`.

## .JuliaFormatter.toml

```toml
style = "sciml"
indent = 4
margin = 92
```

Popular styles: `sciml` (4-space, aligned assignments), `blue` (2-space), `yas` (yet another style).

## src/YourPackageName.jl

```julia
module YourPackageName

# Imports
using StaticArrays

# Include submodules
include("interface.jl")
include("core.jl")

# Exports
export MyType, my_function

end
```

## test/runtests.jl

```julia
using YourPackageName
using Test
using Aqua

@testset "Code quality (Aqua.jl)" begin
    Aqua.test_all(YourPackageName)
end

@testset "YourPackageName.jl" begin
    include("test_core.jl")
end
```

## LICENSE (MIT)

```
MIT License

Copyright (c) 2026 Your Name

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## .gitignore

```
*.jl.*.cov
*.jl.cov
*.jl.mem
/Manifest*.toml
/docs/Manifest*.toml
/docs/build/
```

## Directory Layout

```
YourPackageName.jl/
├── Project.toml
├── LICENSE
├── README.md
├── .gitignore
├── .JuliaFormatter.toml
├── src/
│   └── YourPackageName.jl
├── test/
│   ├── runtests.jl
│   └── test_core.jl
└── .github/
    └── workflows/
        ├── CI.yml
        ├── TagBot.yml
        ├── CompatHelper.yml
        └── julia_formatter.yml
```
