While using Rubocop as a linter/formatter is a no-brainer for most Rails developers these days, it wasn’t always the case. This means that there are probably tons of repositories of applications or gems out there still not benefitting from any linter configuration, and solving this issue might sometimes be cumbersome, especially on old code.

Here’s a little step-by-step guide on how to do it, based on a recent update I made on an open-source lib I was working on, on which some files are more than 10 years old.

_Disclaimer: this guide assumes that the code under scrutiny has a proper test suite, which is a requirement for serene code-refactoring of any nature._

---

## Install Rubocop & relevant extensions
First, I recommend installing the Rubocop gem and a few extensions, by adding them to your Gemfile and running bundle install:
```
rubocop (required)
rubocop-performance
rubocop-rails
rubocop-rspec
```

Each one of these extensions comes with a specific set of cops (=linting rules) addressing specific needs. Check out their respective documentations on the [Rubocop doc](https://docs.rubocop.org/rubocop/1.67/extensions.html), and do not hesitate to add more of them if it suits your needs.

---


## Check for existing .rubocop.yml config, or create one

In my case, there was an existing Rubocop config in the form of a `.rubocop.yml` file, that had not been updated for years and had never been enforced, meaning that most of the cops were out-of-date or just not respected.


In this case, you can add minimal configuration by updating this file as follows:
```
inherit_from: .rubocop_todo.yml

# The behavior of RuboCop can be controlled via the .rubocop.yml
# configuration file. It makes it possible to enable/disable
# certain cops (checks) and to alter their behavior if they accept
# any parameters. The file can be placed either in your home
# directory or in some project directory.
#
# RuboCop will start looking for the configuration file in the directory
# where the inspected file is and continue its way up to the root directory.
#
# See https://docs.rubocop.org/rubocop/configuration

require:
  - rubocop-performance
  - rubocop-rake
  - rubocop-rspec

AllCops:
  TargetRubyVersion: 3.3 # To be changed to suit your needs
  NewCops: enable
  Exclude:
    - tmp/**/*
    - vendor/bundle/**/* # Recommended if you are using Github Actions
  DisplayCopNames: true

# …
# Here comes the config specific to each of the cops. If starting from scratch, you shouldn’t have any.
```

If not already existing, you can simply create this file manually.

You will then need to create an empty `.rubocop_todo.yml` file at the root of your directory for this file to load properly. We will see in an upcoming step what to do with this file.

You can now run `bundle exec rubocop` in your terminal to check that the config file is correctly loaded. In my case, some errors occurred because of a few of the cops mentioned in the config being outdated. I simply followed the prompted instructions to fix these, and ended up with a clean config file.

---


## Run `rubocop —autocorrect`

Now that you have a proper Rubocop config, you can run the `rubocop` command with the `--safe-auto-correct` option in order for the gem to safely refactor every piece of code that doesn’t respect the imported list of cops:
```
bundle exec rubocop --safe-auto-correct

```

_Note: This command might result in a numerous amount of changes, hence I suggest committing your previous changes in the first place in order to be able to separate autocorrection changes from configuration ones in your VCS._

---


## Fix Potential Broken Code

Once the changes are made, ensure no logic has been broken. Run your test suite and inspect the results of the previous command by **carefully reading the diff** before committing anything.

If you need to disable a misbehaving cop, add it to the `.rubocop.yml` file as follows, and re-run the autocorrect command:

```
Style/MultilineBlockChain: # Replace with the cop to be deactivated
  Enabled: false
```

Once satisfied with the results and confident that the code will run, commit your changes before proceeding to the next step.


---

## Whitelist desired cops autocorrectable with rubocop -A

After autocorrecting the safe offenses, you can either stop here or address the unsafe ones. Unsafe cop corrections are more likely to break your code but can sometimes be autocorrected by the gem. To retain only the ones that do not break any logic, follow these steps:

* Run the following command:

```
bundle exec rubocop -A
```
* Run your test suite and identify the changes that broke it
* In Rubocop’s logs, find the precise cops responsible for the changes
* Add them to the list of ignored cops in the `.rubocop_todo.yml` file by running:

```
rubocop --auto-gen-config --only <YourCopName>
```
* Checkout the previously committed version of every file except `.rubocop_todo.yml`
* Repeat the steps until your test suite succeeds

Inspect the results of the previous operation by carefully reading the diff before committing anything. Once satisfied with the results and confident that the code will run, commit your changes.

You have now run unsafe autocorrection on your entire repository while excluding the non-autocorrectable cops by adding them to a ‘TODO’ file, where you can address them later.

---


## Run rubocop --auto-gen-config to complete the .rubocop_todo.yml file

At this stage, we have covered every autocorrection Rubocop can make. However, some offenses might remain. You can either fix them manually or temporarily append them to the `.rubocop_todo.yml` file as a TODO list to handle later. The following command will add all remaining offenses’ associated cops to the file:
```
bundle exec rubocop --auto-gen-config
```

You can now commit these changes as well, and your repository should subsequently be offence-free. 🎉

---



## Bonus: Ignore the Resulting Revisions in Your git blame

Running autocorrection commands can result in hundreds of lines of diff being committed, polluting the `git blame` results for other contributors.
To avoid this, use the `git --ignore-revs-file` feature (documentation [here](https://git-scm.com/docs/git-blame#Documentation/git-blame.txt---ignore-revs-fileltfilegt)), which lists the revision numbers (or commit SHAs) to be ignored by git blame in a designated file, thus not taking them into account when displaying the future.

---


## Bonus #2: Add a CI step (recommended)

The best way to avoid upcoming offenses is to add a CI task running the Rubocop command and preventing anyone from merging code that contains offenses.

If you are using Github Actions, [basic actions](https://github.com/marketplace/actions/rubocop-linter-action) exist to ensure this in the simplest way.
