# Contribution Guide

This file is primarily intended for developers who wish to fork the project and potentially contribute to it. This project follows [Phased Versioning](https://labs.koseka.net/phased-versioning) for its versioning, branching, and release rules, and [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) for its commit messages. So, make sure to read both first before contributing to the project in any way.

## Project Structure

Here are the main directories and files in the project:

```plaintext
.
├── docs/
│   ├── agent-interface.md
│   ├── model-architecture.md
│   ├── open-questions.md
│   ├── system-architecture.md
│   └── web-apps-and-permissions.md
├── img/
├── HANDOFF.md
└── package.json
```

QualiaOS is in its design phase, so the repository holds design documents only. The `docs/` directory is the reference for what is intended, and `docs/open-questions.md` collects the decisions that still need to be made. The kernel, [Kalium](https://github.com/KosekaLabs/kalium), lives in its own repository with its own contribution rules. `HANDOFF.md` records where the work currently stands.

Additionally, the `package.json` file configures the [Node](https://nodejs.org) environment required to run the [Trunk CLI](https://docs.trunk.io/code-quality/overview) metalinter.

## Setting Up the Development Environment

This project uses the [**Trunk Code Quality CLI**](https://docs.trunk.io/code-quality/overview) as an npm package for formatting and linting, and **npm** as a package manager. So, make sure you have **node.js** and **npm** installed on your machine. You can install both of them from the official [node.js](https://nodejs.org) website.

Next, clone the `qualia-os` repository to your local machine and install the development dependencies:

```sh
git clone https://github.com/KosekaLabs/qualia-os.git      # Clones the repository.
cd qualia-os                                                # Moves into the project directory.
npm install                                                 # Installs the development dependencies.
```

Since the **Trunk CLI** is used for formatting, it's best if you disable the _format on save_ option in your editor to avoid potential conflicts with the project's formatting configurations.

If you are using [**Zed**](https://zed.dev), you can locally disable the _format on save_ option of your editor for this project by adding the following line to the `.zed/settings.json` file at the root of the project directory:

```json
{
	"format_on_save": "off"
}
```

If you are using [**VSCode**](https://code.visualstudio.com), you can locally disable the _format on save_ option of your editor for this project by adding the following line to the `.vscode/settings.json` file at the root of the project directory:

```json
{
	"editor.formatOnSave": false
}
```

As for the linting, the project comes with its own linters and configurations, so if you have your own linters installed with custom configurations, you should make sure they don't conflict with the project's linters. You can check the list of linters (and formatters) along with their configurations in the `.trunk/trunk.yaml` file and the `.trunk/configs/` directory.

If you encounter any issues, feel free to open an issue on the project's [GitHub repository](https://github.com/KosekaLabs/qualia-os/issues).

## Linting and Formatting

The linters and formatters work through git hooks, so they will run automatically when you commit changes. However, it's best to also run them manually before committing changes to avoid failing the commit hook.

To make sure the **Trunk CLI** is managing the git hooks, you can run the following command:

```sh
npm run trunk git-hooks sync
```

You can manually run the linters and formatters using the following commands:

```sh
npm run check                                               # Runs linters and formatters on all the changed files.
npm run check -- --all                                      # Runs linters and formatters on all the files in the repository.
```

You can manually format the files using the following commands:

```sh
npm run fmt                                                 # Formats all the changed files.
npm run fmt -- --all                                        # Formats all the files in the repository.
```

## License

Copyright 2026 Amon Rayfa.

This project is licensed under the [Apache License (Version 2.0)](LICENSE).
