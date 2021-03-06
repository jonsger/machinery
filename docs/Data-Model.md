# Machinery Data Model

The Machinery data model represents the system configuration as processed by
Machinery. It consists of two parts that closely correspond to each other:

  * JSON serialization
  * Internal object model

It is assumed that at the beginning, the JSON serialization will be used only
by Machinery, but later other tools may start using it too. It is therefore
important to consider compatibility and extensibility from the start. On the
other hand, it is assumed that the internal object model will be used only by
Machinery and its components/plugins, not by any external code.


## JSON Serialization

JSON serialization is used to store and exchange the system configuration.



### Data Structures

The machinery data model knows two data structures: Arrays and Objects.

Objects hold a set of key-value pairs and the ruby object is a one-to-one mapping of the
according JSON representation.

```
  "os": {
    "name": "openSUSE 13.2 (Harlequin)",
    "version": "13.2 (Harlequin)",
    "architecture": "x86_64"
  }
```

would be mapped to an ruby object which behaves like this, for example:

```
  os["name"] # => "openSUSE 13.2 (Harlequin)"
  os.version # => "13.2 (Harlequin)"
```

Arrays on the other hand are not just a collection of elements (of an arbitrary type) but can also
have attributes defined on the list itself. This can be used to specify whether a list of
packages contains RPM or DEB packages, for example.

The notation for these arrays is a JSON object containing two elements, an attribute JSON object
and an array containing the elements:

```
  "packages": {
    "_attributes": {
      "package_system": "rpm"
    },
    "_elements": [
      {
        "name": "acl",
        "version": "2.2.52",
        "release": "6.4"
      }
    ]
  }
```

The according `packages` array would then wrap the elements:

```
  packages.length # => 1
  packages.first.name # => "acl"
```

and also have accessors defined for the attributes:

```
  packages.package_system # => "rpm"
```

If an Array does not have any attributes the "_attributes" object can be omitted. The "_elements"
array on the other hand has to be set even if it is empty in order to distinguish Objects from
Arrays.

Array elements and Object values can be either primitive values or Arrays or Objects again.

### Structure

At the top level, the data consists of a JSON object. Each key in this object
corresponds to a configuration scope (e.g. repositories, packages, changed
configuration files, etc.). Data under each key is further structured into
JSON objects and arrays as needed.

There is one special key `meta`, which is used to collect meta data from the
whole document and the information in the scope sections.

For example, a JSON document describing software configuration may look like this:

```json
{
  "repositories": {
    "_attributes": {
      "repository_system": "zypp"
    },
    "_elements": [
      {
        "alias": "YaST:Head",
        "name": "YaST:Head",
        "type": "rpm-md",
        "url": "http://download.opensuse.org/repositories/YaST:/Head/openSUSE_12.3/",
        "enabled": true,
        "autorefresh": true,
        "gpgcheck": true,
        "priority": 99
      },
      ...
    ]
  },
  "packages": {
    "_attributes": {
      "package_system": "rpm"
    },
    "_elements": [
      {
        "name": "kernel-default",
        "version": "3.0.101",
        "release": "0.35.1",
        "arch": "x86_64",
        "vendor": "SUSE LINUX Products GmbH, Nuernberg, Germany",
        "checksum": "2a3d5b29179daa1e65e391d0a0c1442d"
      },
      ...
    ]
  },

  "meta": {
    "format_version": 2,
    "repositories": {
      "modified": "2014-02-10T16:10:48Z",
      "hostname": "example.com"
    },
    "packages": {
      "modified": "2014-02-10T16:10:48Z",
      "hostname": "example.com"
    }
  }
}
```

Structure of the data, required and optional parts, etc. will be precisely
specified in Machinery documentation and checked by Machinery during
deserialization. There are, however, two general rules:

  1. Every JSON object can have a `comment` property containing a string. This
     compensates for lack of comments in the JSON format, with an additional
     benefit that the comments are made an explicit part of the data model.

  2. Every JSON object can contain additional properties beside those
     specified in the documentation (“unknown properties”). Data in such
     properties is simply ignored and carried through as opaque. This ensures
     future extensibility.


### Versioning

Each release of Machinery supports a specific version as the current version of
the system description format. This is the version defined as
`CURRENT_FORMAT_VERSION` in the
[`SystemDescription`](https://github.com/SUSE/machinery/blob/master/lib/system_description.rb)
class. The current version is stored in the `format_version` attribute of the
`meta` section of all descriptions written by Machinery.

When reading system descriptions, Machinery supports the current version of the
format. It is the latest version the tool can support. It also supports all
previous versions.

Descriptions written in older formats have to be upgraded to the latest format
version using `machinery upgrade-format DESCRIPTION`. This command uses the
format migrations in `schema/migrations` to transform the old description into
the current format. See the [`Migration`](https://github.com/SUSE/machinery/blob/master/lib/migration.rb)
class for details.

If Machinery is given a description with a version newer than the current
version of the tool, it exits with an error.

Whenever the format is changed in a way that older versions of Machinery can't
read it anymore without the chance of losing data or other misbehavior,
`CURRENT_FORMAT_VERSION` needs to be increased.

The policy of the format version from and end user point of view is documented
in the
[System Description Format](https://github.com/SUSE/machinery/wiki/System-Description-Format#versioning)
documentation.

**Note:** Machinery <= v0.18 used unversioned documents which are no longer
supported.

## Internal Object Model

### Basics

The system configuration is internally represented as a tree of Ruby
objects. Leaf nodes are simple Ruby values (integers, strings, etc.), the
inner nodes are instances of classes derived from `Machinery::Object`, which
provides an `OpenStruct`-like API for setting and getting attributes, and
`Machinery::Array`, which provides an `Array`-like API.

The class-based design provides a natural way to implement behavior shared by
all nodes while having a possibility to override it for some of them. Also,
using the `Machinery::Object` class (instead of pure hashes) makes the object
tree nice to navigate. For example, getting the first package from a list can
be done using methods:

```ruby
package = config.software.packages.first
```

With Ruby hashes, the code would be uglier:

```ruby
package = config["software"]["packages"].first
```

### Root

The root of the tree is a bit special — it is an instance of the
`SystemDescription` class (a subclass of `Machinery::Object`). In addition to
representing the toplevel JSON object, this class contains JSON serialization,
deserialization and validation code.

### Representing Scopes

Each scope is represented by a specific subclass of `Machinery::Scope`. The
scopes are defined as a model class in the `plugins/model` directory. The
model classes define what data objects the scope contains. There are helpers
to define the structure of the data.

See for example the definition of the packages scope:

```ruby
class Package < Machinery::Object
end

class RpmPackage < Package
end

class DpkgPackage < Package
end

class PackagesScope < Machinery::Array
  include Machinery::Scope

  has_attributes :package_system
  has_elements class: DpkgPackage, if: { package_system: "dpkg" }
  has_elements class: RpmPackage, if: { package_system: "rpm" }
end
```

### Serialization into JSON

The object tree is serialized into JSON by the `SystemDescription#to_json`
method. It recusively walks the tree and serializes all the nodes.

### Deserialization from JSON

The object tree is deserialized from JSON by the `SystemDescription.from_json` method.

### File Data

Some scopes contain file data. The files are not serialized to the JSON, but
stored into scope-specific subdirectories of the directory where the system
description is stored. Depending on the type of files they are either stored
as plain files or in a structure of tar archives containing the files.
