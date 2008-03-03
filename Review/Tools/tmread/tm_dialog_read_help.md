# tm_dialog_read

Luke Daley "ld at ldaley dot com"

A replacement for `read()` that invokes `tm_dialog` to get the **stdin** data if there is none waiting. 

The goal of this is to facilitate scripts/commands running via TextMate that require user input to use `tm_dialog` to get the input, where typically the user would enter it on the command line.

## How It Works

The OS X dynamic linker is capable of loading libraries that replace functions defined in libraries that an executable is actually linked against. We take advantage of this by inserting a replacement for the system `read()` function that is responsible for reading data from file descriptors.

Our implementation of `read()` invokes `tm_dialog` to prompt the user for input if stdin does not have any data at the time of the `read()` call. 

## Instructions

There are a few things you need to do to take advantage of this, and there are also certain aspects you can optionally customise.

### Loading The Library

To load our `read()` implementation, you need to instruct `dyld` to link against our library and use it before any others. To do this you need to do 2 things. You need to set the `DYLD_FORCE_FLAT_NAMESPACE` environment variable (not to any particular value, just set it) and to add our library to the `DYLD_INSERT_LIBRARIES` environment variable **before** launching the process that will take advantage of it.

The `DYLD_INSERT_LIBRARIES` variable is particularly funny about empty paths (unlike `PATH` for example), so you need to be a bit careful about setting it.

For example ...

    #!/usr/bin/env bash
    
    export DYLD_FORCE_FLAT_NAMESPACE=

	tm_read=/path/to/tm_dialog_read.dylib
	if [ $DYLD_INSERT_LIBRARIES ]
	then
	    export DYLD_INSERT_LIBRARIES="$tm_read:$DYLD_INSERT_LIBRARIES"
	else
	    export DYLD_INSERT_LIBRARIES="$tm_read"
	fi
     
    groovy somescript.groovy

**Note:** The environment variable `DIALOG` also needs to be set to the path of `tm_dialog`. However, this is automatically set by TextMate, so you typically don't have to worry about it.

### The Nib

The environment variable `DIALOG_NIB` optionally specifies the nib file to use when invoking `tm_dialog`. If this is not set, the `RequestString.nib` shipped with TextMate is used.

`DIALOG_NIB` can either be the full path to a nib file, or the *name* of a nib in `$TM_BUNDLE_SUPPORT/nibs` or `$TM_SUPPORT/nibs`. For example, to use the `RequestSecureString.nib` file shipped with TextMate, set `DIALOG_NIB=RequestSecureString`

The nib passed to `tm_dialog` is expected to have a certain structure. The two TextMate supplied nibs (in `$TM_SUPPORT/nibs`) RequestString.nib and RequestSecureString.nib conform to this structure. Namely ...

* The window title should be bound to `title` of the parameters object.
* A text prompt should be bound to `prompt` of the parameters object.
* A NSTextField (or equivalent) should be bound to `string` of the parameters object.
* Two buttons with values bound to `button1` and `button2` of the parameters dictionary of the file's owner.
    * button1 should have it's action bound to the `returnArgument` method, passing the value of `string`
    * button2 should close the window when clicked, returning nothing.
 
The nib will be loaded with '`Send`' as the value for `button1` and '`Send EOF`' as the value for `button2`.

### Customising The Nib

There are environment variables you can set to customise the window title, prompt and initial value:

* `DIALOG_TITLE`
* `DIALOG_PROMPT`
* `DIALOG_STRING`

### Bugs

There is only one known bug, but it's more of a limitation than a bug. The returned input is limited to the size of the buffer passed to read (which seems to be usually `4096`). This shouldn't be too much of a problem as the buffer is rather large and you wouldn't expect the user to input such a quantity of data. If the user did input more data into the dialog than the buffer can handle, it will be silently truncated.