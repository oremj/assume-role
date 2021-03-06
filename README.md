# assume-role

<img src="./assets/assume-role.png" align="right" alt="assume-role logo" />

Assume IAM roles through an **AWS Bastion** account with **MFA** via the command line.

**AWS Bastion** accounts store only IAM users providing a central, isolated account to manage their credentials and access. Trusting AWS accounts create IAM roles that the Bastion users can assume, to allow a single user access to multiple accounts resources. Under this setup, `assume-role` makes it easier to follow the standard security practices of MFA and short lived credentials.

## Installation

`assume-role` requires [`jq`](https://stedolan.github.io/jq/) and [`aws`](https://aws.amazon.com/cli/) CLI tools to be installed.

### via Homebrew (macOS)

```bash
brew tap coinbase/assume-role
brew install assume-role
```

You can then upgrade at any time by running:

```bash
brew upgrade assume-role
```

### via Bash (Linux/macOS)

You can install/upgrade assume-role with this command:

```bash
curl https://raw.githubusercontent.com/coinbase/assume-role/master/install-assume-role -O
cat install-assume-role # inspect the script for security
bash ./install-assume-role # install assume-role
```

It will ask for your sudo password if necessary.

## Getting Started

Make sure that credentials for your AWS bastion account are stored in `~/.aws/credentials`.

Out of the box you can call `assume-role` like:

```bash
eval $(assume-role account-id role mfa-token)
```

If your shell supports bash functions (e.g. zsh) then you can add `source $(which assume-role)` to your `rc` file (e.g. `~/.zshrc`), then you can call `assume-role` like:

```bash
assume-role [account-id] [role] [mfa-token]
```

`assume-role` this method can be used with arguments or interactively like:

<img src="./assets/assume-role.gif" alt="assume-role usage" />

### Account Aliasing

You can define aliases to account ids in `~/.aws/accounts` which assume-role can use, e.g.

```json
{
  "default": 123456789012,
  "staging": 123456789012,
  "production": 123456789012
}
```

With this file, to assume the `read` role in the `production` account:

```bash
assume-role production read
# OR
assume-role 123456789012 read
```

## AWS Bastion Account Setup

Here is a simple example of how to set up a **Bastion** AWS account with an id `0987654321098` and a **Production** account with the id `123456789012`.

In the **Production** account create a role called `read`, with the trust relationship:

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::0987654321098:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "true",
          "aws:MultiFactorAuthPresent": "true"
        },
        "NumericLessThan": {
          "aws:MultiFactorAuthAge": "54000"
        }
      }
    }
  ]
}
```

The conditions `aws:MultiFactorAuthPresent` and `aws:MultiFactorAuthAge` forces the use of temporary credentials secured with MFA.

In the **Bastion** account, create a group called `assume-read` with the policy:

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [ "sts:AssumeRole" ],
      "Resource": [ "arn:aws:iam::123456789012:role/read" ],
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true",
          "aws:SecureTransport": "true"
        },
        "NumericLessThan": {
          "aws:MultiFactorAuthAge": "54000"
        }
      }
    }
  ]
}
```

Attach this group to **Bastion** users that should be able use `read`'s policies in the **Production** account.

You can assume the `read` role in **Production** by running:

```
assume-role 123456789012 read
```

Then entering a MFA token on request.

## Prompt

If you are using `zsh` you can get a sweet prompt by adding to your `.zshrc` file:

```bash
# AWS ACCOUNT NAME
function aws_account_info {
  [ "$AWS_ACCOUNT_NAME" ] && [ "$AWS_ACCOUNT_ROLE" ] && echo "%F{blue}aws:(%f%F{red}$AWS_ACCOUNT_NAME:$AWS_ACCOUNT_ROLE%f%F{blue})%F$reset_color"
}

# )ofni_tnuocca_swa($ is $(aws_account_info) backwards
PROMPT=`echo $PROMPT | rev | sed 's/ / )ofni_tnuocca_swa($ /'| rev`
```

## Testing

assume-role is tested with [BATS](https://github.com/sstephenson/bats) (Bash Automated Testing System). To run the tests first you will need `bats`, `jq` and `shellcheck` installed. On macOS this can be accomplished with `brew`:

```bash
brew install bats
brew install jq
brew install shellcheck
```

Then run `bats test/assume-role.bats`;
