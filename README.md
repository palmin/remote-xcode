# remote-xcode

Claude Code plugins for working on your Xcode projects from the Claude app on
your phone, while Claude runs on your Mac through Remote Control.

- **deploy-ipa** puts debug builds of your iOS apps on your iPhone without
  TestFlight or a cable, and lets Claude read their console output wherever
  the phone is.
- **edit-link** lets you edit a file of your project by hand on the phone,
  such as release notes or a config file, and brings what you save back
  into the file.

Both keep everything in a private bucket of your own, on AWS S3 or another
S3 service with presigned links, and can share one bucket and key. Nothing is
added to your Xcode projects, and nothing runs on anyone's server but your
own bucket.

## deploy-ipa

Ask Claude to "deploy MyApp to my phone". It archives the scheme, uploads a
development-signed .ipa to your bucket, and replies with a link to a page
with an Install button. You tap the link right there in the conversation on
the phone you test on, and it stays there for later. Ask for "a build that
uploads its logging" instead, and the build also streams its `print` and
`NSLog` output to the bucket, where Claude reads it while you reproduce a bug.

## edit-link

Ask Claude to "let me edit the TestFlight notes". It uploads the file with an
editor page to your bucket and replies with a link. You edit on the phone and
tap Save, and within seconds Claude applies your change to the file as an
ordinary edit, so you see it as a diff in the conversation. When Claude
changes the file meanwhile, an open page shows the new version, and when you
both changed it, the two are merged with conflicts marked, so neither side
overwrites the other.

## What you need

- A Mac running Claude Code, which you drive from the Claude app through
  Remote Control or Claude Code on the web.
- For deploy-ipa: Xcode, using automatic signing, with an Xcode account
  signed in for the team. The phone must be registered with that team, with
  Developer Mode on (Settings > Privacy & Security).
- A private bucket on AWS S3 or another S3 service with presigned links,
  such as Cloudflare R2, Backblaze B2 or MinIO, and an access key for it.
- For deploy-ipa, the `aws` CLI 2.13 or later (`brew install awscli`), `jq`,
  `python3` and `curl`. edit-link needs only `python3`, `git` and `curl`,
  which recent macOS has; `setup` checks.

## Install

In Claude Code, add this repository once, then install either plugin or both:

```
/plugin marketplace add palmin/remote-xcode
/plugin install deploy-ipa@remote-xcode
/plugin install edit-link@remote-xcode
```

Or copy `plugins/<name>/skills/<name>` to `~/.claude/skills/<name>` to use it
as a personal skill.

## Set up

1. **Bucket.** Create a bucket in a region near you, keeping it private (on
   AWS, Block Public Access stays on). On AWS its name must not contain
   dots, as the links use `https://<bucket>.s3.<region>.amazonaws.com`;
   other services are addressed path-style, `<endpoint>/<bucket>/<key>`.
2. **Access key.** Any key that can read, write, list and delete in the
   bucket works. As the links carry its id and the Mac holds its secret,
   limit it to the bucket where the service lets you. On AWS, give it to an
   IAM user with only the policy in [iam-policy.json](iam-policy.json), with
   `YOUR-BUCKET` replaced, leaving out the `DeployIpa` or `EditLink`
   statements of a plugin you don't use, and changing `deploy-ipa` or
   `edit-link` in it if you pick other prefixes.
3. **Run setup in your own Terminal**, for each plugin. Ask Claude to "set
   up deploy-ipa" or "set up edit-link" to get the exact path, which depends
   on how you installed it, then run it yourself:

   ```bash
   <skill folder>/setup
   ```

   It stores the keys in your login Keychain, typed into macOS's own prompt,
   so they never pass through Claude, and asks for the bucket and, for
   other services than AWS, the endpoint. It writes
   `~/.config/deploy-ipa.env` or `~/.config/edit-link.env`. deploy-ipa's
   also copies a small Keychain helper to `~/.config/deploy-ipa/aws-credentials`
   and adds a profile to `~/.aws/config` for the `aws` CLI. With deploy-ipa
   set up, edit-link's setup offers to use the same keys and bucket, so you
   type them only once. Then it checks everything: a signed download, that
   the same link without its signature is refused, and what each plugin
   needs on top (console uploads for deploy-ipa; the change checks saves
   rely on, listing and deleting for edit-link).

Run `setup --check` at any time to repeat the checks without changing
anything; Claude runs it too when something fails.

## Use deploy-ipa

- "Deploy MyApp to my phone", or `/deploy-ipa:deploy-ipa MyApp`
- "Send a Release build of MyApp to my iPhone"
- "Show me the install link again"
- "Make a build of MyApp that uploads its logging"
- "I reproduced it, what did MyApp log?"
- "Watch the MyApp console live while I try this"
- "I'm on the same Wi-Fi, install it directly and watch the console"

Links last 7 days. Console output arrives 2 to 5 seconds late, with the time
and process name on each line, and the app and each of its extensions upload
their own. From a device, only `print` and `NSLog` come through, because
reading the log store for `Logger` lines takes seconds there; so add
temporary debug lines as `NSLog` or `print`.

The Mac must be unlocked while it builds. On a locked Mac, `xcodebuild`
cannot read the Xcode account and signs Xcode out of every developer account,
so deploy-ipa refuses to start; unlock it with Screen Sharing, for instance.

## Use edit-link

- "Let me edit the TestFlight notes on my phone", or `/edit-link:edit-link notes.txt`
- "Give me a link to edit Config.xcconfig"
- "I saved it" or "Did I change anything?"
- "Show me the edit link again"
- "I'm done with the notes"

Links last 7 days. After sharing, Claude waits up to an hour for your saves
and picks each one up within about 3 seconds; after that, tell it when you
saved. The page takes UTF-8 text up to 2 MB, follows dark mode, and keeps
iOS from zooming in. Typing on an iPhone can bring curly quotes, which Claude
points out in code and config files rather than keeping silently.

## What leaves your Mac

- The secret key never does. Install and edit links are presigned URLs,
  which carry the access key id and a signature. They appear in the
  conversation, so anyone who can read the transcript can use them until they
  expire after 7 days: download the build, or read and change the shared
  file. A development-signed .ipa installs only on devices registered with
  your team.
- A console build contains a signed upload policy instead of a key. Anyone
  holding the binary can upload files of up to 10 MB under that build's own
  console folder until the policy expires with the links, but cannot read,
  list or delete anything.
- An edit page holds links that read and replace that one file and nothing
  else. When you are done, Claude stops sharing it, which deletes the copy
  from the bucket.
- Each deploy deletes everything under its prefix older than the link
  lifetime, and each share deletes the shares nothing was saved to for that
  long, so give each plugin a prefix of its own. In a versioned bucket,
  deleted and replaced versions stay until a lifecycle rule removes them.

## Limits

- Everything needs only presigned links, listing and deleting, except
  console builds, which upload with presigned POST. Not every S3 service
  takes that (Cloudflare R2 does not), so `deploy-ipa --console` tries one
  upload first and says so.
- deploy-ipa builds iOS apps only, console builds need iOS 15 or later, and
  builds run one at a time, since schemes share DerivedData.
- edit-link edits UTF-8 text files up to 2 MB.

## Files

| File | Purpose |
|---|---|
| `.claude-plugin/marketplace.json` | lists both plugins for `/plugin marketplace add` |
| `iam-policy.json` | the permissions the IAM user needs, per plugin |
| `plugins/deploy-ipa/skills/deploy-ipa/SKILL.md` | tells Claude how to deploy, show the link and read console output |
| `plugins/deploy-ipa/skills/deploy-ipa/deploy-ipa` | archives, packages, uploads and prints the links |
| `plugins/deploy-ipa/skills/deploy-ipa/deploy-ipa-console` | prints or follows what a console build uploaded |
| `plugins/deploy-ipa/skills/deploy-ipa/build-console-object` | compiles `RemoteConsole.m` with an upload policy |
| `plugins/deploy-ipa/skills/deploy-ipa/upload-policy` | signs the S3 POST policy for console uploads |
| `plugins/deploy-ipa/skills/deploy-ipa/RemoteConsole.m` | captures stdout, stderr, `NSLog` and `Logger` lines and uploads them |
| `plugins/deploy-ipa/skills/deploy-ipa/aws-credentials` | reads the AWS keys from the Keychain for the `aws` CLI |
| `plugins/deploy-ipa/skills/deploy-ipa/setup` | sets deploy-ipa up and checks it |
| `plugins/edit-link/skills/edit-link/SKILL.md` | tells Claude how to share a file, wait for saves and apply them |
| `plugins/edit-link/skills/edit-link/edit-link` | shares, fetches, merges and stops sharing; checks the setup |
| `plugins/edit-link/skills/edit-link/editor.html` | the editor page, uploaded next to each shared file |
| `plugins/edit-link/skills/edit-link/setup` | sets edit-link up and checks it |

## About me

I'm [Anders Borum](https://mastodon.social/@palmin), making developer
tools for iOS and Mac, most notably [Working Copy](https://workingcopy.app),
the full-featured Git client for iOS putting your
side-projects in your side pocket where they belong.
