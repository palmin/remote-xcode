# remote-xcode

Claude Code plugins for working on your Xcode projects from the Claude app on
your phone while you are away from your Mac, with Claude running on the Mac
through Remote Control. On the same Wi-Fi, Xcode already installs and debugs
over the network; these work when the phone is anywhere else, on mobile data
or another network.

- **deploy-ipa** puts debug builds of your iOS apps on your iPhone without
  TestFlight, a cable or a shared network, and lets Claude read their console
  output wherever the phone is.
- **edit-link** lets you edit a file of your project by hand on the phone,
  such as release notes or a config file, and brings what you save back
  into the file.

Both go through a private bucket of your own, which the Mac and the phone
can each reach from anywhere: AWS S3 or another S3 service with presigned
links. Nothing is added to your Xcode projects, and nothing runs on anyone's
server but your own bucket.

In this recording, the Claude app on an iPhone drives Claude Code on a Mac.
Claude deploys a debug build of FTP Files to the phone, then shares the
app's TestFlight notes for editing in Safari and applies the edit to the
file. The waits for the build and for Claude are sped up.

<img width="360" height="778" alt="remote-xcode-demo-360" src="https://github.com/user-attachments/assets/febf2123-c6c5-491c-856e-1806015082b7" />

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
3. **Run setup in your own Terminal** for each plugin. Ask Claude to "set
   up deploy-ipa" or "set up edit-link" for the path, then run:

   ```bash
   <skill folder>/setup
   ```

   It asks for the bucket, the endpoint for services other than AWS, and the
   keys, which you type into macOS's own prompt so they go straight into
   your Keychain and never pass through Claude. Then it checks that
   everything works. edit-link's setup offers to reuse deploy-ipa's keys and
   bucket.

`setup --check` repeats the checks without changing anything; Claude runs it
when something fails.

## Use deploy-ipa

Claude archives the scheme, uploads a development-signed .ipa to your bucket
and replies with a link to an install page, which stays in the conversation.
A build that uploads its logging also streams `print` and `NSLog` output to
the bucket, for Claude to read while you reproduce a bug.

- "Deploy MyApp to my phone", or `/deploy-ipa:deploy-ipa MyApp`
- "Send a Release build of MyApp to my iPhone"
- "Show me the install link again"
- "Make a build of MyApp that uploads its logging"
- "I reproduced it, what did MyApp log?"
- "Watch the MyApp console live while I try this"
- "I'm on the same Wi-Fi, install it directly and watch the console"

Links last 7 days. Console output arrives 2 to 5 seconds late, from the app
and each of its extensions. From a device only `print` and `NSLog` come
through, so add temporary debug lines with those.

The Mac must be unlocked while it builds: on a locked Mac, `xcodebuild` signs
Xcode out of every developer account, so deploy-ipa refuses to start.

## Use edit-link

Claude uploads the file with an editor page to your bucket and replies with
the link. When you tap Save, Claude applies your change to the file as an
ordinary edit, so you see it as a diff in the conversation. When the file
also changed on the Mac, the two are merged with conflicts marked, so
neither side overwrites the other.

- "Let me edit the TestFlight notes on my phone", or `/edit-link:edit-link notes.txt`
- "Give me a link to edit Config.xcconfig"
- "I saved it" or "Did I change anything?"
- "Show me the edit link again"
- "I'm done with the notes"

Links last 7 days. For an hour after sharing, Claude picks up each save
within seconds; after that, tell it when you saved.

## What leaves your Mac

- The secret key never does. Install and edit links carry only the key id
  and a signature, and anyone who can read the conversation can use them
  until they expire after 7 days. An edit link reaches only its one file.
- A console build holds a signed upload policy, not a key: anyone with the
  binary can upload up to 10 MB into that build's console folder until it
  expires, but not read, list or delete anything.
- Old builds and unused shares are deleted after 7 days, and a share also as
  soon as you stop it, so give each plugin a prefix of its own. A versioned
  bucket keeps old versions until a lifecycle rule removes them.

## Limits

- Everything needs only presigned links, listing and deleting, except
  console builds, which upload with presigned POST. Not every S3 service
  takes that (Cloudflare R2 does not), so `deploy-ipa --console` tries one
  upload first and says so.
- deploy-ipa builds iOS apps only, console builds need iOS 15 or later, and
  builds run one at a time, since schemes share DerivedData.
- edit-link edits UTF-8 text files up to 2 MB.

## About me

I'm [Anders Borum](https://mastodon.social/@palmin), making developer
tools for iOS and Mac, most notably [Working Copy](https://workingcopy.app),
the full-featured Git client for iOS putting your
side-projects in your side pocket where they belong.
