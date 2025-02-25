# Continuous Integration

## GitHub Actions - `Bower`-based

```yml
name: CI

# Run CI when a PR is opened against the branch `main`
# and when one pushes a commit to `main`.
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# Run CI on all 3 latest OSes
jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, macOS-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v2

      - uses: purescript-contrib/setup-purescript@main
        with:
          purescript: "0.15.7"
          purs-tidy: "0.9.2"
          psa: "0.8.2"

      - uses: actions/setup-node@v
        with:
          node-version: "16"

      - name: Install dependencies
        run: |
          npm install -g bower
          npm install
          bower install --production

      # Compile the library/project
      #   censor-lib: ignore warnings emitted by dependencies
      #   strict: convert warnings into errors
      - name: Build source
        run: |
          pulp build -- --censor-lib --strict

      - name: Run tests
        run: |
          bower install
          pulp test

      - name: Check Formatting
        run: |
          purs-tidy check src test
```

## GitHub Actions - `Spago`-based

```yml
name: CI

# Run CI when a PR is opened against the branch `main`
# and when one pushes a commit to `main`.
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# Run CI on all 3 latest OSes
jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v3

      - name: Set up Node toolchain
        uses: actions/setup-node@v3
        with:
          node-version: "16"

      - name: Cache NPM dependencies
        uses: actions/cache@v3
        env:
          cache-name: cache-node-modules
        with:
          path: ~/.npm
          key: ${{ runner.os }}-build-${{ env.cache-name }}-${{ hashFiles('**/package.json') }}
          restore-keys: |
            ${{ runner.os }}-build-${{ env.cache-name }}-
            ${{ runner.os }}-build-
            ${{ runner.os }}-

      - name: Setup PureScript tooling
        run:
          npm i -g purescript@latest purs-tidy@latest purescript-psa@latest spago@latest

      - name: Install NPM dependencies
        run: npm install

      - name: Cache PureScript dependencies
        uses: actions/cache@v3
        with:
          key: ${{ runner.os }}-spago-${{ hashFiles('**/*.dhall') }}
          path: |
            .spago
            output

      # Compile the library/project
      #   censor-lib: ignore warnings emitted by dependencies
      #   strict: convert warnings into errors
      # Note: `purs-args` actually forwards these args to `psa`
      - name: Build the project
        run: |
          npx spago build --purs-args "--censor-lib --strict"

      - name: Run tests
        run: |
          npx spago test

      - name: Check Formatting
        if: runner.os == 'Linux'
        run: |
          purs-tidy check src test
```
