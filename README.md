# IBM Installation Manager Cookbook

## Scope
This cookbook installs IBM Installation Manager and provides resources for installing IBM packages with a response file.

This cookbook currently uses the Installation Manager Installer method rather than Installation manager to install packages.  For more details see http://www-01.ibm.com/support/knowledgecenter/SSDV2W_1.8.4/com.ibm.silentinstall12.doc/topics/c_installing_differences.html

Group install mode has not been tested. Only admin and nonAdmin have.

## Requirements
* Chef 12.5 or higher
* Java needs to be pre-installed. The libraries do not install Java intentionally so it can me managed separately in your wrapper.
* Installation Media for Installation manager and any other packages.
This can be sourced from an online repository like IBM PassportAdvantage, or from your own download link.  Supports tar, tar.gz and zip.
* Installation Kits may need to be packaged into a single folder.  For example the Websphere installation kit comes in 3 folders disk1, disk2 and disk3.  Make sure all disks are in the same folder like so:

  - WAS/
  - WAS/disk1/
  - WAS/disk2/
  - WAS/disk3/
  - WAS/repository.config

## Platforms
* Centos 6
* Red Hat Enterprise 6
* Centos 7
* Red Hat Enterprise 7

## Usage

See the test cookbook in test/fixtures/cookbooks/ibm-im-test for examples.

##### Install Installation Manager
```ruby
# install java
node.default['java']['jdk_version'] = '7'
node.default['java']['install_flavor'] = 'oracle'
node.default['java']['oracle']['accept_oracle_download_terms'] = true
include_recipe 'java'

install_mgr 'ibm-im install' do
  install_package 'https://myrepository/agent.installer.linux.gtk.x86_64_1.8.4000.20151125_0201.zip'
  install_package_sha256 '28f5279abc28695c0b99ae0c3fdee26bfec131186f2ca7e41d1317e303adb12e'
  package_name 'com.ibm.cic.agent'
  ibm_root_dir '/opt/IBM'
  service_user 'ibm-im'
  service_group 'ibm-im'
end

ibm_secure_storage_file '/root/MySecureStorageFile' do
  master_pw_file '/root/MyMasterPassFile'
  master_pw 'mypassphrase'
  passport_advantage true
  username 'myPassportAdvUser'
  password 'mypassword'
end

ibm_package 'IHS install' do
  packages ['com.ibm.websphere.IHS.v85_8.5.5000.20130514_1044']
  install_dir '/opt/IBM/WAS'
  passport_advantage true
  imcl_dir '/opt/IBM/InstallationManager/eclipse/tools'
  master_pw_file '/root/MyMasterPassFile'
  action :install
end

```

##### Install Websphere with Installation Manager

```ruby
# installation manager must be installed before calling this resource
ibm_package 'WAS ND install' do
  packages ['com.ibm.websphere.ND.v85']
  install_dir '/opt/IBM/WAS'
  repositories ['/opt/ibm-media/WASND']
  imcl_dir '/opt/IBM/InstallationManager/eclipse/tools'
  action :install
end
```

### Resources


### install_mgr

Installs IBM Installation Manager

##### Example
```ruby
install_mgr 'ibm-im install' do
  install_package 'https://some-repository/agent.installer.linux.gtk.x86_64_1.8.4000.20151125_0201.zip'
  install_package_sha256 '28f5279abc28695c0b99ae0c3fdee26bfec131186f2ca7e41d1317e303adb12e'
  package_name 'com.ibm.cic.agent'
  ibm_root_dir '/opt/IBM'
  service_user 'ibm-im'
  service_group 'ibm-im'
end
```

##### Parameters

- `:install_package`, Url or local path to a compressed file. Must be .tar, .tar.gz or .zip. String, required: true, default: nil
- `:install_package_sha256`, Hash of install package. String, default: nil
- `:download_temp_dir`, Scratch folder to download to. String, default: Chef::Config['file_cache_path']
- `:extract_dir`, String, default: ""#{download_temp_dir}/ibm-installmgr"
- `:install_dir`, String, default: "#{ibm_root_dir}/InstallationManager/eclipse"
- `:package_name`, String, default: 'com.ibm.cic.agent'
- `:repositories`, [String, Array], default: [extract_dir]
- `:ibm_root_dir`, String, default: '/opt/IBM'
- `:data_location`, This is where Installation Manager stores data about it's installed packages. String, default: '/var/ibm/InstallationManager'
- `:service_user`, String, default: 'ibm-im'
- `:service_group`, String, default: 'ibm-im'
- `:access_rights`, String, default: 'nonAdmin', Can only be nonAdmin, admin or group.  Group is not tested.
- `:preferences`, An additional string to add preferences. String, default: 'offering.service.repositories.areUsed=false'

##### Actions

- `:install` - Installs Installation Manager. Does not start it as this requires gui.


### ibm_package

Installs an IBM application using Installation Manager

##### Example
```ruby
ibm_package 'WAS ND install' do
  packages ['com.ibm.websphere.ND.v85']
  install_dir '/opt/IBM/WAS'
  repositories ['http://repo/WASNDv85.tar.gz']
  imcl_dir '/opt/IBM/InstallationManager/eclipse/tools'
  action :install
end
```

##### Parameters

- `:packages`, An array of application id's to install. Eg ['com.ibm.websphere.ND.v85_8.5.5000.20130514_1044'] [String, Array] required: true, default: nil
- `install_dir`, Where to install the package. String, required: true, default: nil
- `imcl_dir`, Path to the imcl utility binary You shouldn't need to change this. String, default: '/opt/IBM/InstallationManager/eclipse/tools'
- `repositories`, Array of local dirs or urls to repositories.  Installation Kits may need to be packaged into a single folder, see the Requirements section above. [String, Array], required: true, default: nil
- `service_user`, String, default: 'ibm'
- `service_group`, String, default: 'ibm'
- `additional_options`, a string of additional options to append if needed. String, default: ''
- `access_rights`, String, default: 'nonAdmin', regex: /^(nonAdmin|admin|group)$/
- `log_dir`, String, default: '/var/ibm/InstallationManager/logs'
- `:passport_advantage`, When set to true adds passportAdvantafe repo url to repositories. default: false
- `:secure_storage_file`, path to secure storage file to use for repo credentials. String, default: nil
- `:master_pw_file`, path to file containing password to access encrypted secure_storage_file. String, default: nil

##### Actions

- `:install` - Installs the package. Does not start, setup a service at this stage.


### ibm_response_file

A simple wrapper around the template resource to create a response file for the ibm_package_response resource. You can use the template resource directly if you like.

##### Example
```ruby
ibm_response_file '/opt/IBM/response_files/my_was_response.xml' do
  cookbook 'ibm-im-test'
  template_source 'was_response.xml.erb'
  variables({
    :key1 => 'val1'
  })
end

```

##### Parameters

- `response_file`, Path to create the response file. String, name_property: true
- `group`, File group. String, default: 'ibm-im'
- `owner`, File owner. String, default: 'ibm-im'
- `template_source`, String, default: nil
- `cookbook`, String, default: nil
- `variables`, [Hash], default: nil

##### Actions

- `:create`


### ibm_package_response

Installs an IBM package using a response file.
Please note you may need to create and required users and directories mentioned in the response file first.  You do not need to accept the license or set the accessRights in the response file, this is handled with the command.

##### Example
```ruby
ibm_package_response '/opt/IBM/response_files/my_was_response.xml' do
  action :install
end

```

##### Parameters

- `response_file`, String, name_property: true, required: true
- `imcl_dir`, String, default: '/opt/IBM/InstallationManager/eclipse/tools'
- `log_dir`, String, default: '/var/ibm/InstallationManager/logs'
- `group`, Only used to for log_dir. String, default: 'ibm-im'
- `owner`, Only used to for log_dir String, default: 'ibm-im'
- `access_rights`, String, default: 'nonAdmin', regex: /^(nonAdmin|admin|group)$/

##### Actions

- `:install`


### ibm_secure_storage_file

Creates an encrypted Eclipse Storage file to store password credentials to repositories such as IBM passport Advantage.

##### Example
```ruby
ibm_secure_storage_file '/root/MySecureStorageFile' do
  master_pw_file '/root/MyMasterPassFile'
  master_pw 'mypassphrase'
  passport_advantage true
  username 'myemail@somedomain.com'
  password 'mypass'
end
```

##### Parameters

- `secure_file`, String, name_property: true
- `master_pw_file`, Contains the password to access the secure_file. String
- `master_pw`, password to put in master_pw_file. String, default: nil
- `url`, String, default: 'http://www.ibm.com/software/repositorymanager/entitled/repository.xml'
- `passport_advantage`, Set to true when credentials are to access passportAdvantage. The url will be ignored if set. [TrueClass, FalseClass], default: false
- `username`, String, default: nil
- `password`, For encrypting the secure file. String, default: nil
- `imutilsc_dir`, Path to imutilsc binary. You shouldn't need to change this from the default. String, default: '/opt/IBM/InstallationManager/eclipse/tools'
##### Actions

- `:create`


## License and Author

* Author: Lachlan Munro, Chris Bell

Copyright: 2015, J Sainsburys

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.