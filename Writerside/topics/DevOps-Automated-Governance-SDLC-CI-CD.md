# (UNDER CONSTRUCTION) - DevOps Automated Governance - SDLC (CI/CD)

Your organization may required, either by Policy or a Control Framework, an assurance process for all changes to production. Therefore, their is a requirement to verify key details of the Software Development Lifecycle (SDLC): the Continuous Integration (CI) and/or Continuous Deployment (CD) processes. 

Every so often, someone with a title like "GRC", "Compliance Manager", or "Internal Auditor" comes by your desk once a yar to ask for the evidence that we followed the SDLC portion of our change management process for a handful of changes we made about 13 or 14 months ago. Playing this game is always a pain in the butt because we have to dig through our numerous tools and systems to re-discover what we already know; we followed our process.

We can solve this headache with the NAPE Collection CLI.  How can we use the CLI help?

First, we will use the Collection CLI to collect the evidence from our Peer Review tool which is where we track who requests and approves merge requests into the source code branch we use for production.  Second, we will the NAPE Procedures to automates the [test of details](Glossary.topic#test-of-details) which prove we are doing the two specific things we are supposed to do:

1) Ensure the person requesting the merge does not approve the merge.
2) Ensure there are, at least, two (2) people who reviewed the change before it's merged into the production branch.

We have many other things as well we need to capture, such as Unit Testing, Integration Testing, Common Vulnerability & Exploits (CVE), and Common Weaknesses & Enumerations (CWEs).  For right now, we are going to automate the evidence collection and test of details for the Peer Review step of our CI process.

## Assumptions

Assuming this is our first use of the Collection CLI, we need to set two (2) things up: an [Evidence Repository](Evidence-Repository.md), and a [NAPE Repository](NAPE-Repository.md)

- **Evidence Repository**
    - The CLI connects to either AWS S3, GCP Blob Storage, or Azure Blog Storage to store collected evidence & compliance reports.
    - You can get detailed information for their configuration in the [Collection CLI Configuration](collection-cli-configuration.md).

- **NAPE Repository**
    - The CLI configuration requires a link to the git repository where you are storing your NAPE Procedures for the test of details.
    - If the repository is private, make sure to provide the proper credentials as outlined in the CLI Configuration Section

## Pass, Fail, & Inconclusive

[NAPE Procedures](NAPE-Procedures.md) is how we codify the [test of details](Glossary.topic#test-of-details) which verify if a single [control activity](Glossary.topic#control-activity) either passed, failed, or was inconclusive.

Test of details are whats reffereed as Substantive Test, Thee test substantiate that something took place.  It's a common term used in the formal audit practices.

A ***PASS*** outcome happens when the data that NAPE is looking for exists in the evidence, and the data is either equal to or within a set of expected bounds.

A ***FAIL*** outcome happens when the data that NAPE is looking for exists in the evidence, and the data is not equal to or outside a set of expected bounds.

The ***INCONCLUSIVE*** outcome happens when NAPE cannot find the data it's looking for in your evidence.  This is important to understand because the inconclusive outcomes tell you that what you are looking to measure was not provided in the data.  NAPE only fails when it can find the data and the data is not as expected.  Therefore, there is no information to tell us if the activity was a pass or fail, therefore an inconclusive determination is made.

## Using The Collection CLI

Below are the CLI Commands in the order of the process.  This does not relate to any specific CI tool implementation, although it can be used as a guide regardless of your CI tooling.

For high level awareness, we will run four (4) commands. Each command will be explained more in-depth in subsequent section:

{type="narrow"}
`collect start`
: Starts the process of collecting evidence and associating the evidence with the source code commit.

`collect evidence`
: Will collect the evidence you tell it to.

`collect artifact`
: Record an artifact was created during the process and obtains the artifacts signature.

`collect finish`
: Will evaluate all the evidence, generate a report of the test of details, and upload the report to the Evidence Repository.

## 1st Command - **`new`**
### Starting the Collection Process

The first thing you must always do is establish that you are starting a process that you want to provide assurance around.

To do this, we use the `collect start` command.

```bash
$ collect start /
    -subject nrn:sourcecode:[group]:[repository-name] /
    -subject-id [git-commit-sha-1] /
    -process [link-to-process-description] /
    -meta build-id [build-id] /
```

Here is A practical example, using GitHub Actions as CI.

```bash
$ collect start /
    -subject nrn:sourcecode:nape:collection-cli /
    -subject-id $GITHUB_SHA / # For subsequent examples, assume this value is "9f3f183a300501b53e2fa04f48acb4bd478d6414"
    -process-location github.com/nape-not-another-policy-engine/nape-catalog.git
    -meta build-id $GITHUB_RUN_ATTEMPT # For subsequent examples, assume this value is "1"

```
Before we get too far down the road, let's talk about the `nrn` you see for the `-subject` flag.  NRN standards for [NAPE Resource Name](NRNs.topic) and this is a specific way to identify unique things.  An NRN is a specific format that abides by the [Universal Resource Name (URN)](https://www.issn.org/services/online-services/urn/) conventions.

The NRN you see in this example, `nrn:sourcecode:nape:collection-cli` is composed of 4 aspects, which follows the convention of `nrn:sourcecode:[group]:[repository-name]`

- `nrn` - This signifier represents this URN is an NAPE Resource Name.
- `sourcecode` - This is called Namespace Identifier (NID) and identifies what follows as an NRN which represents a source code repository
- `[group]` - Group represents ownership of this NRN.  This is an arbitrary value you provide that helps you identify who owns the resources the NRN refers to.  This is referred to as a Namespace String (NSS).
- `[repository-name]` - Repository Name represents an arbitrary value that you use to identify the name of your sourcecode repository.  This is also a NSS.

The sourcecode NRN in now way tries to mimic the URL of your repository. It is recommended that you keep the sourcecode NRN as similar to  your sourcecode repository URL to reduce misunderstandings, although you do not need to.  Some simple examples are provided below when we talk more about the `-subject` flag.

### What does the `collect start` command do?

The purpose of the `collect start` command is to:

1. Set up the local directory structure to store copies of evidence, test procedures, and the process description.
2. Create the report file `assurance_report.yaml` where all the information is captured about the process outcomes.
3. Download all the process and procedure files from the `-process` location.

Let's get detailed with what happens when you invoke the `collect start` command.

#### First, Create the `./nape` Directory

A directory name `./nape` is created in the present working directory of your CI tool. Some CI tools may share the same disk space, so the CLI ensures that there are no conflicts by always creating a new directory for this specific run. If the directory already exits, the CLI moves to its next step

#### Second, Create a Subdirectory for the CI Run

A subdirectory for this specific process run is created inside the `./nape` directory. This new directory be named based on a modified version of the `subject`. Within the `subject` directory, a subdirectory will be created that is the first seven (7) characters of the commit-sha from the `subject-id`.  Inside the `sugject-id` directory there wil be another subdirectory that is the `build-id`.

Using the practical example above for the GitHub action, assume the following:

- `$GITHUB_SHA` = 9f3f183a300501b53e2fa04f48acb4bd478d6414
- `$GITHUB_RUN_ATTEMPT` = 1

The full directory where data about the run will be stored is: `.nape/nrn_sourcecode_nape---collection-_-cli/9f3f183/1`.

#### Third, Populate the CI Run Directory

Inside the directory named after the `[build-id]`, the following actions will take place:

- A base report file name `-repornapet.yaml` is created.
- The `-process` is downloaded into the `./temp` directory.
- The `process.yml` is moved from the `./process` directory into the same directory as the `nape-report.yaml`.
- Three additional subdirectories will be created:
    - `/evidence` - This directory will contain a copy of all the evidence you collect in future steps.
    - `/procedures` - This directory is where the NAPE procedures listed in your process field will be copied for local use.
    - `/temp` - This directory is where all filess are downloaded to before they are moved into another directory.

Continuing from our *NAPE Collection CLI* example above, this is what you would expect to see in a directory structure as follows:

```text
root
└─── nrn_sourcecode_nape---collection-_-cli
    |
    └─── 9f3f183
            |   assurance_process.yml
            |
            └─── activity
            |
            └─── evidence
            |
            └─── temp
```

Let's learn more about the `assurnace_report.yaml`, `assurnace_procedure.yaml`, and `./activity`.

##### `assurnace_report.yaml`

This document contains the links and records to all the data used in assessing the assurance of this CI run.  The purpose of this document is to have a single comprehensive record of:

- What was analyzed,
- How it was analyzed, and
- A summary compilation of all the results.

Using our *NAPE Collection CLI* example above, the initial `assurnace_report.yaml` document would look like the following:

```yaml
---
apiVersion: v1.0.0
kind: AssuranceReport
metadata:
  build-id: "1"
subject:
  nrn: "nrn:sourcecode:nape:collection-cli"
  id: "9f3f183a300501b53e2fa04f48acb4bd478d6414"
process: 
  location: "github.com/nape-not-another-policy-engine/processes/rust-ci/sourcecode-integration"
```

##### `process.yaml`

This document is the assurance process specification which defines:

- the list of control procedures,
- the expected evidence files,
- the activities which identify the [test of details](Glossary.topic#test-of-details) to apply to a specific evidence file.

Let's dig into an abbreviated example.  If you need more insights, [check out full `process.yaml` document specification (ProcedureDefinition)](ProcedureDefinition.md) with in-depth explanations.

Below is the `process.yaml` document that was downloaded from <github.com/nape-not-another-policy-engine/processes/rust-ci/sourcecode-integration> :

```yaml
---
apiVersion: v1.0.0
kind: ProcedureDefinition
process:
  nrn: "nrn:process:nape/software-processes:rust-ci/sourcecode-integration"
  short: "Rust CI Control Activities"
  description: |
    "This is the complete process required to integrate a source code change for a rust application 
    back into the trunk, or main branch, of the source code repository."
procedures:
  - name: "peer-review"
    short: "Peer Review"
    description: "This procedure verifies that a peer review was completed to expectation"
    expect-evidence:
      - "./evidence/peer-review/peer-review.json"
    activity:
      - name: "at-least-two-reviewers"
        short: "Two (2) Reviewer Approval"
        description: "There are at least two (2) peer reviews who approved the merge request."
        test: "./procedures/peer-review/at-least-two-reviewers.py"
        evidence: "./evidence/peer-review/peer-review.json"
      - name: "requester-not-a-reviewer"
        short: "Requester is not Approver"
        description: "The person who initiated the merge request is not one of the people who approved the merge request."
        test: "./procedures/peer-review/requester-not-a-reviewer.py"
        evidence: "./evidence/peer-review/peer-review.json"
  - name: ...
artifacts:
    - name: "runtime-binary"
      description: "This is the software program customers will use."
      with-metadata:
        - "artifact-repo"
```

**`./procedures`:**

The CLI will download the `../rust-ci/sourcecode-integration/process.yaml` and the `../rust-ci/sourcecode-integration/procedures` directory from <github.com/nape-not-another-policy-engine/processes/rust-ci/sourcecode-integration> repository and copy the procedures into your local `./procedures` directory.

At this time, everything has been set up for you to start adding evidence

### Understanding the `collect start` Attributes

#### `-subject`

The `subject` NAPE term for the "thing" that is being assessed for compliance. The NAPE Resource Name `nrn` is the special Universal Resource Name (URN) specific to nape resources.  You can [find out more about nrn's here](https://www.todo-this-link.com). Inside this `nrn` is the reserved Namespace Identifier (NID) of `sourcecode`. the NID of `sourcecode` signifies that everything related to this ARTN has to do with the software source code. The `[group]` is a Namespace-Specific String (NSS) is an arbitrary value that describes the owner of your source code repo, and the `[repository-name]` is an arbitrary value that describes the name of your sourcecode repository.

Here are some examples of how we recommend you generate the NSS for different source code repositories:

- **GitHub**
    - Repo Link -  https://www.github.com/nape-not-another-policy-engine/collection-cli
    - NRN - `nrn:sourcecode:nape:collection-cli`
- **GitLab**
    - Repo Link -  https://www.gitlab.com/nape-not-another-policy-engine/ui-project/frontend
    - NRN - `nrn:sourcecode:nape/ui-project:frontend`

#### `-subject-id`

The`subject-id` is a place to provide a business-specific unique identifier for this instance of the process execution.  This identifier is used to refer back to your systems of record when necessary. For your CI process, ***we recommend the build number supplied by your CI tooling***. You can provide your unique ID as well if you feel it's necessary.

***Why the build number?*** In the background, we collect the GIT SHA of the commit and the date & time for this specific run. This build number is used in instances when the same GIT SHA has more than one CI process executed against it.

Why would there be more than one CI process executed for any given commit SHA? It's not uncommon to re-run the CI process for a given commit SHA because previous CI runs failed or execution was halted for a variety of reasons.

#### `-process`

This `-process` flag is a network-accessible location to download an [NAPE Process Definition](ProcedureDefinition.md) and its related control activities.  The process definition outlines all the procedures and activities you want to verify that took place based on the evidence you have collected.

#### `-meta`

The `-meta` flag allows you to provide metadata for any step that is run.  Based on the subject ARN's Namespace Identifier (NID), specific metadata will be required.

For the `nrn:sourcecode`, the git SHA-1 is always required using the key `commit-sha`

You can learn more about the required `-meta` flags for other NRNs at the [NAPE Required NRN Metadata](NRNs.topic)

## 2nd Command - **`evidence`**
### Collect Evidence for Peer Review

Let's check that a peer review was completed. To do that we need to get the data to show there was a peer review, and save it to a file.

Here is a sudo-code example of what this may look like all together:

```bash
## Code To Retrieve Peer Review Document
curl -o downloaded-report.json http://www.my-internal-peer-review-tool.com/the/peer/review.json

##  Use the Collect CLI to copy and label the evidence
collect evidence /
    -procedure peer-review
    -file ./downloaded-report.json peer-review.json
```

Let's dig into the details of this step.

### First, Retrieve Peer Review Data

The first thing we do is reach out to whichever tool we use to track peer review data and download the data as evidence.  When we download the data, we can choose any arbitrary name to save it as.

### Second, Invoke `collect evidence`

The `evidence` command takes the local data you want to use as evidence and copies it into the `./evidence/peer-review` directory using the name `peer-review.json` for the file.  **The file name we upload must match one of the expected file names** in the peer review procedure which is outlined in the `ProcedureDefinition`.

The `-file [0] [1]` flag takes in two arguments.  The first argument is the location of the file to collect.  The second argument is the name of the file if you need it renamed when it's collected. If you upload a file whose name is not included in any of the corresponding `expected-evidence` file names, then any `activity` that requires this file will be marked as **INCONCLUSIVE** when it comes time to assess the control activities.

When there is a need to upload more than one file as evidence for a procedure, then you simply repeat the `-file` attribute until you have collected all the appropriate evidence.

Below is the example `peer-review` from the `ProcedureDefinition` above. You'll see that for the peer review procedure, we expect to have evidence named `peer-review.json`, which is why we used `peer-review.json` as the second argument.

```yaml
---
apiVersion: v1.0.0
kind: ProcedureDefinition
...
procedures:
  - name: "peer-review"
    short: "Peer Review"
    description: "This procedure verifies that a peer review was completed to expectation"
    expect-evidence:
      - "./evidence/peer-review/peer-review.json"
    activity:
      - name: "at-least-two-reviewers"
        short: "Two (2) Reviewer Approval"
        description: "There are at least two (2) peer reviews who approved the merge request."
        test: "./procedures/peer-review/at-least-two-reviewers.py"
        evidence: "./evidence/peer-review/peer-review.json"
      - name: "requester-not-a-reviewer"
        short: "Requester is not Approver"
        description: "The person who initiated the merge request is not one of the people who approved the merge request."
        test: "./procedures/peer-review/requester-not-a-reviewer.py"
        evidence: "./evidence/peer-review/peer-review.json"
...
```

At this point, we have the following directory structure with the associated files:

```text
root
└─── .nape
    |
    └─── nrn_sourcecode_nape---collection-_-cli
        |
        └─── 9f3f183
            |
            └─── 1
                |   attesitfy-report.yaml
                |   process.yml
                |
                └─── evidence
                  └─── peer-review
                      |   peer-review.json
                |
                └─── procedures
                  └─── peer-review
                      | at-least-two-reviewers.py
                      | requestor-not-a-reviewer.py
                  | ...
                |
                └─── temp
                  | ...
```

#### What Can Error During This Command?

Three conditions can go wrong when this command is invoked.  Below are the conditions and what will happen.

1. The file you're adding as evidence does not exist.
    1. A warning will be written to standard out stating that the file you are attempting to upload does not exist.
2. You have supplied an improper file name to rename the file when collected.
    1. A warning will be written to standard out stating the file name is incorrect.
    2. The fill will still be collected but with its current name.
3. The procedure you referenced is not in the `ProcedureDefinition` an info statement will be written to standard out describing that you are collecting evidence for a procedure that does not exist.
    1. The file will be collected and put into a directory within the `./evidence` directory which corresponds to the `-procedure` name you provided.
    2. Since this procedure does not exist in your process, it will not be evaluated.
    3. It will appear on the report as `additional-information`

When it comes time to assess activities associated with a file that is missing, each activity will be evaluated as **INCONCLUSIVE** meaning there wasn't enough information to determine if the activity was a pass or fail.

## 3rd Command - **`artifact`**
### Associate A Build Artifact with Process

The ultimate question someone is going to ask is, "Prove to me that this [binary, library, service, ect...] met all compliance expectations."  The ability to answer this with one-click and a smile is why NAPE was created.
When you add an artifact, you are stating that this artifact(s) is an outcome the process you are providing assurances for.

While we have no explicit build step in this tutorial, let's pretend that after the Peer Review stage, we create a build of our application. This build could be an executable binary, or individual file, that includes, but is not limited to, a *.jar, *.exe, *.js, ect.  The build can also include a directory of stuff, such as a JavaScript, Python, or a Web Site.

To associate an artifact with your current process run, use the following command:

```bash
collect artifact /
    -name "runtime-binary"
    -location ./link/to/dir/or/file
    -meta artifact-repo "https://some-repo.com/my-artifact"
```

### What Happens `-collect artifact` Runs

The Collection CLI will add the artifact to the `artifacts` section of the `AssuranceReport`

Continuing with our example, an `artifacts` section will be added to the `assurance_report.yaml`.  Inside this section, each artifact you record.

```yaml
---
apiVersion: v1.0.0
kind: AssuranceReport
...
procedures: ...
artifacts:
  - name: "runtime-binary"
    signature: [some-sha-256-value]
    meta:
      artifact-repo: "https://some-repo.com/my-artifact"
    issues: ...
```

You can record as many artifacts as you'd like.  If there are expected artifacts outlined in the `ProcedureDefinition`, then when the `collect finish` command is invoked, it will evaluate all artifacts to ensure they were provided properly.

#### What Can Error During the `collect artifact` Command?

One (1) conditions can go wrong when this command is invoked.  Below are the conditions and what will happen:

1. The artifact file or directory you are referencing does not exist.
    1. A warning will be written to standard out stating that the file, or directory, does not exist.
    2. The artifact will not be recorded
    3. An statement will be added to the `issues` section of the `AssuranceReport` stating this artifact was not processable because the file/directory provided does not exist.

## 4th Command - **`report`**
### Run Test of Details and Create the Assurance Report

We are about done.  So far, we have started a new process, and we have collected our evidence for our Peer Review procedure.  The only thing left is to execute all the of the test of details and generate the report.

> **Ensure `collect finish` Is Invoked After CI Script Errors**
>
> It's always paramount to use `collect finish` in two spots in your CI script.
>
> - **#1** - Use `collect finish` where you expect your CI or CD process to come to a logical end; assuming all previous commands and process ran successfully.
> - **#2** - Always ensure have `collect finish` invoked on you CI hooks that run after an error or termination is throw in your CI script.  If not, your CI process will end and your evidence collection may be incomplete, and your report will not be executed.
>
> Additionally, you can use [`collect info`](collect-info.md) command to log additional information that is not related to evidence.  For example, if a scan or tool throws a non-zero exit code to signal it fails, you can write all the log information to a file then add it to your reports so you have easy access to read about the failure details before the CI run stops processing.
>
{style="tip"}

To start the reporting phase, invoke the `collect finish` phase as follows:

```bash
collect finish
```

### What Happens `-collect finish` Runs

The Collection CLI goes through each procedure in the `process.yaml` and begins to invoke each test of detail for each of the listed activities.  Once the test of detail has ran, the CLI then updates the `assurance_report.yaml` with the outcome result.

Below is what the `assurance_report.yaml` looks like once all the peer-review activities have been executed.  You can learn the specifics of the assurance report by view the [AssuranceReport Reference](AssuranceReport.md)

```yaml
---
apiVersion: v1.0.0
kind: AssuranceReport
metadata:
  build-id: "1"
subject:
  nrn: "nrn:sourcecode:nape:collection-cli"
  id: "9f3f183a300501b53e2fa04f48acb4bd478d641"
process: 
  location: "github.com/nape-catalog/processes/rust-ci/sourcecode-integration"
summary:
    activity-count: 2
    activity-run: 2
    pass: 1
    fail: 1
    inconclusive: 0
    outcome: fail
    artifact-count: 1
    artifacts-expected: 1
procedures:
  - name: peer-review
    activity:
      - name: at-least-two-reviewers
        outcome: fail
        reasons: "There are only one (1) Reviewer(s): ['other_user'], when there should be at least two (2)."
        test:
          file: "./procedures/peer-review/at-least-two-reviewers.py"
          signature: "61f9c779b71c376ac789dd37282aabc5636dcac391be73f49292b42c94aa5392"
        evidence:
          file: "./evidence/peer-review/peer-review.json"
          signature: "2e45228ed9b3b8e2a9312c99b72163bbbc3d8c5973d2afe17dd1e6e60ed93e03"
      - name: requester-not-a-reviewer
        outcome: pass
        reason: "The PR Requester is not a reviewer. Requester: [octocat], Reviewer(s): ['other_user']"
        test:
          file: "./procedures/peer-review/requester-not-a-reviewer.py"
          signature: "bc6a0250cd71ca61c7e795a3a972b3b7735cca5c94a645b51e749f3005cd441e"
        evidence:
          file: "./evidence/peer-review/peer-review.json"
          signature: "d2a03f142f66f519f73187cb03a48929cf521eb76a21da70010f3d00b65e4c57"
  - name: ...
artifacts:
  - name: "runtime-binary"
    signature: [some-sha-256-value]
    meta:
      artifact-repo: "https://some-repo.com/my-artifact"
additional-information: ...
issues: ...
```
### What Can Error During The `collect finish` Command?

Three conditions can go wrong when this command is invoked.  Below are the conditions and what will happen.

1. The `expected-evidence` evidence file does not exist.
    1. A warning will be written to standard out stating that the file you are attempting to test does not exist.
    1. The test will be recorded as **INCONCLUSIVE**
    1. A reason will be provided to the effect of: *"The expected evidence was not found: [file-name]"*
1. The `test` file does not exist.
    1. A warning will be written to standard out stating tha the test file does not exist.
    1. The test will be recorded as  **INCONCLUSIVE**
    1. A reason will be provided to the effect of: *"The expected test was not found: [file-name]"*
1. The evidence and test files exist, but an execution failure happens during the evaluation of a test.
    1. A warning will be written to standard out stating there was an execution failure.
    1. The test will be recorded as  **INCONCLUSIVE**
    1. Details of the execution failure will be listed in the reason file.
1. An artifact identified in the `ProcedureDefinition` was not provide
    1. A warning will be written to standard out stating which metadata was expected that was not provided.
    1. An statement will be added to the `issues` section stating which metadata was expected that was not provided.
    1. This artifact will NOT BE included in the `artifact-count`
1. The expected artifact metadata was not provided.
    1. A warning will be written to standard out stating which artifact metadata was expected that was not provided.
    1. An statement will be added to the `issues` section of the specific artifact in then `artifacts` section stating which metadata was expected that was not provided.
    2. The artifact WILL BE included in the `artifact-count`
1. A error prevents the transmission of the report to the Evidence Repository
    1. A warning will be written to standard out stating there was an upload error, and it will provide the details.

## Check Your Evidence Repository

That's it, it's that simple to get started!  To verify everything has been submitted, simply log into either AWS, GCP, or Azure and check the blob storage configured for your CLI.  Which ever directory you configured as the root, you'll see the following directory structure:
nrn_sourcecode_nape
```text
root
└─── ---collection-_-cli
    |
    └─── 9f3f183
        |
        └─── 1
            |   assurance_report.yaml
            |   process.yml
            |
            └─── evidence
              └─── peer-review
                  |   peer-review.json
            |
            └─── procedures
              └─── peer-review
                  | at-least-two-reviewers.py
                  | requestor-not-a-reviewer.py
              | ...
```

Each and every subject will have its own directory. The immediate subdirectory of the subject will be the instance of your CI run.  Inside the instance will be the assurance report, process definition, and the directories containing the evidence and a copy of the procedures utilized.

## Wrapping It All Up

You now have the basic approach for capturing evidence and providing assurance during your CI or CD process.  All you need to do now is simply add more `collect evidence` commands where you need them and create any corresponding [NAPE Procedures](NAPE-Procedures.md) and [ProcedureDefinitions](ProcedureDefinition.md) and store them in a [NAPE Repository](NAPE-Repository.md).