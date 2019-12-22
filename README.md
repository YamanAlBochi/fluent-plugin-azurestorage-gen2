# Azure Datalake Storage Gen2 Fluentd Output Plugin

[![Build Status](https://travis-ci.org/oleewere/fluent-plugin-azurestorage-gen2.svg?branch=master)](https://travis-ci.org/oleewere/fluent-plugin-azurestorage-gen2)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Gem Version](https://badge.fury.io/rb/fluent-plugin-azurestorage-gen2.svg)](http://badge.fury.io/rb/fluent-plugin-azurestorage-gen2)
![](https://ruby-gem-downloads-badge.herokuapp.com/fluent-plugin-azurestorage-gen2?extension=png)

## Requirements

| fluent-plugin-azurestorage-gen2 | fluentd | ruby |
|------------------------|---------|------|
| >= 0.1.1 | >= v0.14.0 | >= 2.4 |

## Overview

Fluent output plugin that can use ABFS api and append blobs with MSI and OAuth support.

## Installation

Install from RubyGems:
```
$ gem install fluent-plugin-azurestorage-gen2
```

## Configuration

- Configuration on VMs by using MSI:
```
<match **>
  @type azurestorage_gen2
  azure_storage_account            mystorageabfs
  azure_container                  mycontainer
  azure_instance_msi               /subscriptions/mysubscriptionid
  azure_object_key_format          %{path}-%{index}.%{file_extension}
  azure_oauth_refresh_interval     3600
  time_slice_format                %Y%m%d-%H
  file_extension                   log
  path                             "/cluster-logs/myfolder/${tag[1]}-#{Socket.gethostname}-%M"
  auto_create_container            true
  <buffer tag,time>
    @type file
    path /var/log/fluent/azurestorage-buffer
    timekey 5m
    timekey_wait 0s
    timekey_use_utc true
    chunk_limit_size 64m
  </buffer>
</match>
```

- Configuration outside of VMs with OAuth credentials:
```
<match **>
  @type azurestorage_gen2
  azure_storage_account            mystorageabfs
  azure_container                  mycontainer
  azure_object_key_format          %{path}-%{index}.%{file_extension}
  azure_oauth_tenant_id            <my tenant id>
  azure_oauth_app_id               <my app client id>
  azure_oauth_secret               <my client secret>
  azure_oauth_refresh_interval     3600
  time_slice_format                %Y%m%d-%H
  file_extension                   log
  path                             "/cluster-logs/myfolder/${tag[1]}-#{Socket.gethostname}-%M"
  auto_create_container            true
  <buffer tag,time>
    @type file
    path /var/log/fluent/azurestorage-buffer
    timekey 5m
    timekey_wait 0s
    timekey_use_utc true
    chunk_limit_size 64m
  </buffer>
</match>
```

### Configuration options

### azure_storage_account

Your Azure Storage Account Name. This can be got from Azure Management potal.
This parameter is required when environment variable 'AZURE_STORAGE_ACCOUNT' is not set.

### azure_storage_access_key (not implemented yet - use msi)

Your Azure Storage Access Key(Primary or Secondary). This also can be got from Azure Management potal. Storage access key authentication is used when this parameter is provided or environment variable 'AZURE_STORAGE_ACCESS_KEY' is set.

### azure_instance_msi

Your Azure Managed Service Identity ID. When storage key authentication is not used, the plugin uses OAuth2 to authenticate as given MSI. This authentication method only works on Azure VM. If the VM has only one MSI assigned, this parameter becomes optional and the only MSI will be used. Otherwise this parameter is required.

### azure_oauth_tenant_id

Azure account tenant id from your Azure Directory. Required if OAuth based credential mechanism is used.

### azure_oauth_app_id

OAuth client id that is used for OAuth based authentication. Required if OAuth based credential mechanism is used.

### azure_oauth_secret

OAuth client secret that is used for OAuth based authentication. Required if OAuth based credential mechanism is used.

### azure_oauth_refresh_interval

OAuth2 access token refreshment interval in second. Only applies when MSI / OAuth authentication is used.

### azure_container (Required)

Azure Storage Container name

### auto_create_container

This plugin create container if not exist when you set 'auto_create_container' to true.

### azure_object_key_format

The format of Azure Storage object keys. You can use several built-in variables:

- %{path}
- %{time_slice}
- %{index}
- %{file_extension}

to decide keys dynamically.

%{path} is exactly the value of *path* configured in the configuration file. E.g., "logs/" in the example configuration above.
%{time_slice} is the time-slice in text that are formatted with *time_slice_format*.
%{index} is the sequential number starts from 0, increments when multiple files are uploaded to Azure Storage in the same time slice.
%{file_extention} is always "gz" for now.

The default format is "%{path}%{time_slice}_%{index}.%{file_extension}".

For instance, using the example configuration above, actual object keys on Azure Storage will be something like:

```
"logs/20130111-22_0.gz"
"logs/20130111-23_0.gz"
"logs/20130111-23_1.gz"
"logs/20130112-00_0.gz"
```

With the configuration:

```
azure_object_key_format %{path}/events/ts=%{time_slice}/events_%{index}.%{file_extension}
path log
time_slice_format %Y%m%d-%H
```

You get:

```
"log/events/ts=20130111-22/events_0.gz"
"log/events/ts=20130111-23/events_0.gz"
"log/events/ts=20130111-23/events_1.gz"
"log/events/ts=20130112-00/events_0.gz"
```

The [fluent-mixin-config-placeholders](https://github.com/tagomoris/fluent-mixin-config-placeholders) mixin is also incorporated, so additional variables such as %{hostname}, %{uuid}, etc. can be used in the azure_object_key_format. This could prove useful in preventing filename conflicts when writing from multiple servers.

```
azure_object_key_format %{path}/events/ts=%{time_slice}/events_%{index}-%{hostname}.%{file_extension}
```

### format

Change one line format in the Azure Storage object. Supported formats are 'out_file', 'json', 'ltsv' and 'single_value'.

- out_file (default)

```
time\ttag\t{..json1..}
time\ttag\t{..json2..}
...
```

- json

```
{..json1..}
{..json2..}
...
```

At this format, "time" and "tag" are omitted.
But you can set these information to the record by setting "include_tag_key" / "tag_key" and "include_time_key" / "time_key" option.
If you set following configuration in AzureStorage output:

```
format json
include_time_key true
time_key log_time # default is time
```

then the record has log_time field.

```
{"log_time":"time string",...}
```

- ltsv

```
key1:value1\tkey2:value2
key1:value1\tkey2:value2
...
```

"ltsv" format also accepts "include_xxx" related options. See "json" section.

- single_value

Use specified value instead of entire recode. If you get '{"message":"my log"}', then contents are

```
my log1
my log2
...
```

You can change key name by "message_key" option.

### path

path prefix of the files on Azure Storage. Default is "" (no prefix).

### buffer_path (required)

path prefix of the files to buffer logs.

### time_slice_format

Format of the time used as the file name. Default is '%Y%m%d'. Use '%Y%m%d%H' to split files hourly.

### time_slice_wait

The time to wait old logs. Default is 10 minutes.

### utc

Use UTC instead of local time.

## TODOs

- add storage key support
- add compression (if append is not used)

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
