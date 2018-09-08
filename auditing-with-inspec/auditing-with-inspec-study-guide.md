# Auditing with InSpec

## General notes

The exam felt easy and short, but that is probably because I have been working
with InSpec quite a lot. There are different details popping up, which you
might overlook if you are just studying, but not practicing.

Currently, the InSpec documentation is not yet well-structured - so some points
are easy to overlook. It already has gotten significantly better compared to
the state two years ago though :)

## Exam

Same style as any of the recent Chef exams: Remotely proctores via PSI and you
get a WebVNC-Style Ubuntu machine to work on. In this case, you already get a
test machine to work with.

The exam consists of a multiple choice part within that workstation, as far as
I remember also with questions of the like "Which kernel is this machine
running on". The second half is a performance based part where you are asked
to create an InSpec profile with different properties (I only hint you at
profile inheritance here to stay fair to everyone).

The test machine is set up with Linux as well, so no Windows in this exam. In
my opinion, all questions were straight and clear. No word play or gibberish.

## External Resources

* [Exam information](https://training.chef.io/auditing-with-inspec-badge)
  * [Exam blueprint (PDF)](https://training.chef.io/static/Auditing_with_InSpec_Badge_Scope.pdf)
* All items under Reference in the [InSpec documentation](https://www.inspec.io/docs/)
* On [learn.chef.io](learn.chef.io)
  * "Compliance Automation with InSpec"
  * "Integrated Compliance with Chef" (parts 1 + 4)
* [Annie Hedgpeth's InSpec Day 1 - 10](http://www.anniehedgie.com/inspec-basics-1)

## Study Notes

### Installing InSpec

Execution model of InSpec
  * No software required on node to test, execute via SSH or WinRM
  * Needs InSpec on the scanning node (of course)
  * Ruby is executed on the scanning machine only, not the target
  * for execution on target systems:
    * inspec.command('...')
    * inspec.powershell('...')
    * inspec.file('...')

Installation can be done via different ways
  * via ChefDK (it's included)
  * via [InSpec packages](https://downloads.chef.io/inspec)
  * via ```curl https://omnitruck.chef.io/install.sh | sudo bash -s -- -P inspec```
  * via APT repository
    ```
    apt-get install apt-transport-https
    wget -qO - https://packages.chef.io/chef.asc | apt-key add -
    echo "deb https://packages.chef.io/repos/apt/<CHANNEL> <DISTRIBUTION> main" > /etc/apt/sources.list.d/chef-<CHANNEL>.list
    apt-get update && apt-get install inspec
    ```
  * via ```gem install inspec```

### Running InSpec

* ```inspec detect``` will display information about the local system
* ```inspec detect -t ssh://user@host``` will do the same for remote ones
* Target specifications
  * ```ssh://<user>@<host``` with keys
  * ```ssh://<user>:<pass>@<host>``` with good old passwords
  * ```docker://<container-id>``` against Docker containers
  * ```winrm://<user>@<host> -p <password>``` for Windows machines
  * Out of scope for exam
    * ```aws://<region>``` for inspecting AWS configuration
    * ```azure://``` for Azure
* Fetchers (where to get the profile)
  * ```/local/directory``` well, duh!
  * ```https://some.server/url``` will accept .zip/.tar.gz, see ```inspec archive```
      You can point this against Github/Bitbucket and it will rewrite the URL to the .tar.gz file
  * ```supermarket://dev-sec/os-hardening``` via InSpec extension (bundled with main InSpec)
* Arguments
  * ```--controls control-01 control-02``` for only executing specific controls
  * ```--format json``` (old) and ```--reporter json``` (new)
  * ```--json-config file``` for supplying parameters via file, not CLI
  * Exit codes
  * ```--disctinct-exit``` (default): 100: controls skipped, 101: failed; 0 ok
  * ```--no-distinct-exit``` only 0: ok, 1: fail

### InSpec Profiles

* inspec.yml: contains name, title, description, supports, depends and others
  * format of supports:
    ```
    supports:
     - platform-name: ubuntu
       release: 14.04
    ```
  * format of depends
    * for remote profiles (url: http://...)
    * for local profiles (path: /some/path)
    * for git repos (git:/branch:/tag:/...)
    * for supermarket (supermarket: user/profile)
    * for Chef Compliance (compliance: base/linux)
* semantic versioning, like with Chef, revisit the version comparators like "~>"
* inspec.lock: contains the evaluated dependencies, can be updates with ```inspec vendor --overwrite```
* directory libraries/ is for any Ruby libraries to be loaded/used
* directory files/ can be used for profile specific files
  * load them inside via ```inspec.profile.file(name)```
  * can be used as settings:  ```yaml(content: inspec.profile.file('settings.yaml')).params```
* Controls syntax with:
  * tag (without values)
  * tag (with value)
  * ref
  * impact (between 0.0 and 1.0 now)

* Resources, e.g.
  * command (with the stdout/stderr/exit_status properties)
  * docker
  * file / directory
  * group
  * http (matching responses and status codes)
  * package
  * mssql_session / mysql_session / postgres_session (SQL statements, matching rows and contents)
  * service
  * sshd_config (checking for settings)
  * user

* Creating Custom Resources __TODO__

* Profiles Inheritance to Overlay Custom Changes
  * Two ways: include all but skip some, require another profile and selectivly use some
    ```
    include_controls 'profile' do
      skip_control 'u-05'
    end

    require_controls 'profile' do
      control 'u-05'
    end
    ```
  * Can change attributes of inherited controls:
    ```
    require_controls 'profile' do
      control 'u-05' do
        impact 0.3
      end
    end
    ```

* Profiles Inheritance to Use Custom Resources
  * custom resources from inherited profiles are available __by default__
  * can rename/alias inherited resources
    ```
    require_resource(profile: 'base-profile', resource: 'example', as: 'base-example')
    ```

* Profile Attributes
  * provide configurability of profiles, valid within a ruby file (not the profile)
  * ```level = attribute('level', default: 1, description: 'CIS Level to use')```
  * can be provided as argument to ```inspec exec```:
    ```--attrs=file1.json file2.json``` (multiple files possible, syntax feels a bit awkward)
  * possible use case: supply terraform output to inspec for checks

* Testing Either/Or Configuration Checks
  * If only one alternative needs to be valid
    ```
    describe.one do
      describe '...' do
        it { should cmp 1 }
      end
      describe '...' do
        it { should cmp 2 }
      end
    end
    ```
  * no support yet for "describe.two" or similar, Issues exist
  * can conditionally execute resources, e.g based on an attribute. If not executed, no output
    ```
    if level == 2
      control 'Checks for AuditD' do
        title 'AuditD is required for Level 2'
        desc 'See CIS benchmark'
        impact 0.8
        describe package('auditd') do
          it { should be_installed }
        end
      end
    end
    ```
  * can also be used with only_if for having a "Skip" if not applicable (control level, not single describe)
    ```
    control 'Check unattended_upgrades' do
      only_if { os.debian? }
      describe package('unattended_upgrades') do
        if { should be_installed }
      end
    end
    ```

### Controls and Metadata

* impact is in [0.0 .. 1.0] and signifies criticality of a control, not effort or likeliness
* tag can be just a name or a key/value pair to add metadata (CIS level, OS applicable, ...). Completely freeform
* ref is for external references like ISO standards, CIS benchmarks or the likes

### Troubleshooting:

Mainly via [InSpec Shell](https://www.inspec.io/docs/reference/shell/)

* ```inspec shell``` is an interactive shell to develop or debug InSpec
* can be used local or remotely (same -t flags as with inspec, e.g. -t ssh://root@192.168.0.1)
* is based on [pry](https://github.com/pry/pry)
* core resources are loaded automatically
* fancy one-liner to get methods of resources:
  ```resource('os').class.superclass.instance_methods(false).sort``` where "os" is an inspec resource
