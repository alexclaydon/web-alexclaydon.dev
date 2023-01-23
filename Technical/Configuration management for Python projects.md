## The problem

It's usually necessary when writing any non-trivial Python code to implement some way to easily configure the way it's interpreted. This post on the tools I reach for to achieve this. As I'm writing this for myself, it's opinionated. On the other hand, I hope it's also helpful to anyone else coming at configuration management for the first time and wondering how to handle it.

The approach depends on the type of code you're writing and the intended deployment environment (the [Twelve-Factor App](https://alexclaydon.dev/post/2020-08-09-python-config-and-envs)([https://12factor.net/](https://12factor.net/) is an excellent starting point in this regard), but I've tried to settle on some sensible defaults. Most of the below assumes you are on Ubuntu (or any flavour of Linux, really) and have direct access to the command line.

Before we kick off, it's probably helpful to distinguish configuration management, for secrets management. This post deals strictly with configuration management. Secrets management deserves a separate post. Even though in substance you may end up using (parts of) the same stack to handle both configuration and secrets, the two have very different requirements. We'll look at secrets management (perhaps using Mozilla SOPS or Hashicorp Vault) at some point in the future. I'll be sure to propose an approach there that integrates cleanly with the configuration paradigm described here.

There are potentially a number of different approaches to managing configuration.

### In-code variables

Stored directly in the code itself. This may be convenient during development - or for variables in a production deployment that rarely change. As they will be checked into version control, you will have trivial access to earlier settings. But there downsides, too. It can be hard to make changes once in production, and you have to be confident that none of the configuration settings are potentially sensitive (keeping in mind of course that we're excluding secrets from the current discussion).

Accordingly, to the extent possible, you should avoid keeping any configuration data that is either sensitive, or expected to change, in-code. On the other hand, you shouldn't be discouraged from keeping configuration that is relatively stable over time in your code itself - there's nothing inherently wrong with this, as long as you understand your production needs.

### Environment variables

Environment variables (envs) are variables that are present in your execution environment - be that bare metal, a VM, or - for example - a Heroku deployment environment.

These can be:
- set manually in a bash session from the command line (`MY_KEY='my_value'`) and then imported into your execution environment using `os.getenv()`. Note, however, that envs set in this manner are available only to the execution context of the terminal session where you set them (and any child processes spawned from that session). This may be fine if you're manually running Python scripts from the command line, but is unhelpful for most other execution contexts. In addition, envs set in this manner are ephemeral - they will be lost at the end of the session.
- sourced by bash automatically from any file loaded into the run-time environment (for example, a user's `~/.profile` file)[^1](https://alexclaydon.dev/post/2020-08-09-python-config-and-envs) then loaded into Python using `os.getenv()`. This is better than setting envs manually at the command line in the sense that (depending on where you put them) they may be loaded at the initialisation of each session. On the other hand, you need to ensure that those envs are in fact stored in the correct file for the execution context you are targeting - there's no point storing them in a file that isn't sourced by bash in the context you are targeting. For example, if you are using `cron` to schedule the running of a script, simply placing any relevant envs in your `~/.profile` file won't alone be sufficient.
- sourced from a file using an `.env` extension (usually located in the project folder itself) with the [python-dotenv](https://pypi.org/project/python-dotenv/) package using `load_dotenv()` then `os.getenv()`. This is a tidy solution in that you don't need to think through potentially convoluted Linux user session details. Assuming the file is checked into version control, it travels with the rest of the repo if your deployment context changes, but you will need to ensure that it doesn't contain any sensitive configuration details (and, of course, this approach is not suitable for storing secrets - although you may be able to do that if you opt not to add the file to your Git repo).

### Arbitrary configuration files
Configuration can also be loaded from YAML (or JSON or INI) files: there's nothing special about these file formats (other than, perhaps, their popularity). Any flat file arranged in pairs of keys and values will do, as long as there's a suitable Python library for importing it at run-time (namely, as a hash map or `dict()`). Of the three, I find YAML is the most human-readable but that's obviously subjective.

Two potential downsides to using this approach are that it will expose any sensitive configuration settings (assuming it's stored in the repo alongside the rest of your code), and that you'll have to write the import code yourself, including any code to provide for a hierarcy of precedence between any configuration settings in the file that conflict with any other settings elsewhere (for example, if a given setting is set both using environment variables and in the imported `dict()`).

### Passed at the command line

Configuration can also be passed by the user invoking the code as command line arguments and options at run-time. Obviously this paradigm is most relevant to those writing code intended to be run directly by a human sitting in front of a machine. On the other hand, you may have some code that is sometimes run by a machine, and sometimes by a human, and clearly in those scenarios you will need to provide that any arguments and options passed at the command line override any existing, conflicting settings established by way of environment variables or configuration files.

## The solution

For most cases (and setting aside the needs of a CLI for the moment), my preference is to store configuration settings as environment variables - sourcing them from either or both of the execution context (for example, your `~/.profile` file) and an `.env` file using python-dotenv. Note that envs sourced from an `.env` file using python-dotenv can be configured to either respect or override existing system envs by passing the `override=True` flag to `load_dotenv()`. In this way, you can seamlessly draw on both sources of envs in a transparent way and with clear behaviour. The python-dotenv library is also a great fit with Django.

On the other hand, if for some reason you want to support the use of other configuration files (YAML, JSON, INI) alongside some mix of environment variables, you may want to reach for something which provides a clear precedence hierarchy between those different sources: we need to make sure that the interpreter knows which variable should take precedence when there are conflicts.

Rather than writing custom code to handle this, I'm going to do something here that's usually against my better judgement, and recommend a small and relatively new open source project called [gila](https://gitlab.com/dashwav/gila). As per the project page, the following are some of the key features:

> -   Allow default values to be set for each config key
> -   Automatically find config files on multiple paths
> -   Load in environment variables automatically that have a specific prefix
> -   Support most popular config languages: yaml, toml, json, properties files, hcl, dotenv
> -   Singleton pattern available for ease of use in most applications
> -   Hierarchical loading for non-ambiguous results.

Gila allows you to corral all of your various different bits of configuration into one place and establish a clear order of precedence. Gila documentation is [here](https://gila.readthedocs.io/en/latest/index.html).

[^1](https://alexclaydon.dev/post/2020-08-09-python-config-and-envs): See the official Ubuntu documentation [here](https://help.ubuntu.com/community/EnvironmentVariables#System-wide_environment_variables) for a more thorough discussion on where to store envs on a Linux.