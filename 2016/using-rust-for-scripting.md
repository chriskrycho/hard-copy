---
Title: Using Rust for ‘Scripting’
Subtitle: "With a bonus: cross-compiling from macOS to Windows!"
Date: 2016-11-14 22:00
Category: Tech
Tags: rust, software development, programming languages
Summary: >
    Why I might use Rust instead of Python, with walkthroughs for building a simple "script"-like program and a guide for cross-compiling Rust code to Windows from macOS.
Modified: 2016-11-15 09:00
---

<i class=editorial>**Edit**: fixed some typos, cleaned up implementation a bit based on feedback around the internet.</i>

## I. Using Rust Instead of Python

A friend asked me today about writing a little script to do a simple conversion of the names of some files in a nested set of directories. Everything with one file extension needed to get another file extension. After asking if it was the kind of thing where he had time to and/or wanted to learn how to do it himself (always important when someone has expressed that interest more generally), I said, "Why don't I do this in Rust?"

Now, given the description, you might think, <i class=thought>Wouldn't it make more sense to do that in Python or Perl or even just a shell script?</i> And the answer would be: it depends—on what the target operating system is, for example, and what the person's current setup is. I knew, for example, that my friend is running Windows, which means he doesn't have Python or Perl installed. I'm not a huge fan of either batch scripts or PowerShell (and I don't know either of them all that well, either).

I could have asked him to install Python. But, on reflection, I thought: <i class=thought>Why would I do that? I can write this in Rust.</i>

Writing it in Rust means I can compile it and hand it to him, and he can run it. And that's it. As wonderful as they are, the fact that languages like Python, Perl, Ruby, JavaScript, etc. require having the runtime bundled up with them makes just shipping a tool a lot harder—*especially* on systems which aren't a Unix derivative and don't have them installed by default. (Yes, I know that *mostly* means Windows, but it doesn't *solely* mean Windows. And, more importantly: the vast majority of the desktop-type computers in the world *still run Windows*. So that's a big reason all by itself.)

So there's the justification for shipping a compiled binary. Why Rust specifically? Well, because I'm a fanboy. (But I'm a fanboy because Rust often gives you roughly the feel of using a high-level language like Python, but lets you ship standalone binaries. The same is true of a variety of other languages, too, like Haskell; but Rust is the one I know and like right now.)

<i class=editorial>**Edit the second:** this is getting a lot of views from Hacker News, and it's worth note: I'm not actually advocating that everyone stop using shell scripts for this kind of thing. I'm simply noting that it's *possible* (and sometimes even *nice*) to be able to do this kind of thing in Rust, cross-compile it, and just ship it. And hey, types are nice when you're trying to do more sophisticated things than I'm doing here! Also, for those worried about running untrusted binaries: I handed my friend the code, and would happily teach him how to build it.</i>

## II. Building a Simple "Script"

Building a "script"-style tool in Rust is pretty easy, gladly. I'll walk through exactly what I did to create this "script"-like tool for my friend. My goal here was to rename every file in a directory from `*.cha` to `*.txt`.

1. [Install Rust.][install]

2. Create a new binary:

    ```sh
    cargo new --bin rename-it
    ```

3. Add the dependencies to the Cargo.toml file. I used the [glob] crate for finding all the `.cha` files and the [clap] crate for argument parsing.

    ```toml
    [package]
    name = "rename-it"
    version = "0.1.0"
    authors = ["Chris Krycho <chris@chriskrycho.com>"]

    [dependencies]
    clap = "2.15.0"
    glob = "0.2"
    ```

4. Add the actual implementation to the `main.rs` file (iterating till you get it the way you want, of course).

    ```rust
    extern crate clap;
    extern crate glob;

    use glob::glob;
    use std::fs;

    use clap::{Arg, App, AppSettings};


    fn main() {
      let path_arg_name = "path";
      let args = App::new("cha-to-txt")
        .about("Rename .cha to .txt")
        .setting(AppSettings::ArgRequiredElseHelp)
        .arg(Arg::with_name(path_arg_name)
          .help("path to the top directory with .cha files"))
        .get_matches();

      let path = args.value_of(path_arg_name)
        .expect("You didn't supply a path");
      let search = String::from(path) + "/**/*.cha";
      let paths = glob(&search)
        .expect("Could not find paths in glob")
        .map(|p| p.expect("Bad individual path in glob"));

      for path in paths {
        match fs::rename(&path, &path.with_extension("txt")) {
          Ok(_) => (),
          Err(reason) => panic!("{}", reason),
        };
      }
    }
    ```

5. Compile it.

    ```sh
    cargo build --release
    ```

6. Copy the executable to hand to a friend.

[install]: https://www.rust-lang.org/en-US/downloads.html

In my case, I actually added in the step of *recompiling* it on Windows after doing all the development on macOS. This is one of the real pieces of magic with Rust: you can *easily* write cross-platform code. The combination of Cargo and native-compiled-code makes it super easy to write this kind of thing—and, honestly, easier to do so in a cross-platform way than it would be with a traditional scripting language.[^haskell]

But what's really delightful is that we can do better. I don't even need to install Rust on Windows to compile a Rust binary for Windows.

## III. Cross-Compiling to Windows from macOS

Once again, let's do this step by step. Three notes: First, I got pretty much everything other than the first and last steps here from WindowsBunny on the [#rust] <abbr>IRC</abbr> channel. (If you've never hopped into #rust, you should: it's amazing.) Second, you'll need a Windows installation to make this work, as you'll need some libraries. (That's a pain, but it's a one-time pain.) Third, this is the setup for doing in on macOS Sierra; steps may look a little different on an earlier version of macOS or on Linux.

[#rust]: https://botbot.me/mozilla/rust/

1. Install the Windows compilation target with `rustup`.

    ```sh
    rustup target add x86_64-pc-windows-msvc
    ```

2. Install the required linker ([`lld`][lld]) by way of installing the LLVM toolchain.

    ```sh
    brew install llvm
    ```

3. Create a symlink somewhere on your `PATH` to the newly installed linker, specifically with the name `link.exe`. I have `~/bin` on my `PATH` for just this kind of thing, so I can do that like so:

    ```sh
    ln -s /usr/local/opt/llvm/bin/lld-link ~/bin/link.exe
    ```

    (We have to do this because the Rust compiler [specifically goes looking for `link.exe` on non-Windows targets][rustc-msvc].)

4. Copy the target files for the Windows build to link against. Those are in these directories, where `<something>` will be a number like `10586.0` or similar (you should pick the highest one if there is more than one):

    - `C:\Program Files\Windows Kits\10\Lib\10.0.<something>\ucrt\x64`
    - `C:\Program Files\Windows Kits\10\Lib\10.0.<something>\um\x64`
    - `C:\Program Files (x86)\Microsoft Visual Studio 14.0\VC\lib\amd64`

    Note that if you don't already have <abbr>MSVC</abbr> installed, you'll need to install it. If you don't have Visual Studio installed on a Windows machine *at all*, you can do that by using the links [here][msvc]. Otherwise, on Windows, go to **Add/Remove Programs** and opting to Modify the Visual Studio installation. There, you can choose to add the C++ tools to the installation.

    Note also that if you're building for 32-bit Windows you'll want to grab *those* libraries instead of the 64-bit libraries.

5. Set the `LIB` environment variable to include those paths and build the program. Let's say you put them in something like `/Users/chris/lib/windows` (which is where I put mine). Your Cargo invocation will look like this:

    ```sh
    env LIB="/Users/chris/lib/windows/ucrt/x64/;/Users/chris/lib/windows/um/x64/;/Users/chris/lib/windows/VC_lib/amd64/" \
    cargo build --release --target=x86_64-pc-windows-msvc
    ```

    Note that the final `/` on each path and the enclosing quotation marks are all important!

6. Copy the binary to hand to a friend, without ever having had to leave your Mac.

To be sure, there was a little extra work involved in getting cross-compilation set up. (This is the kind of thing I'd love to see further automated with `rustup` in 2017!) But what we have at the end is pretty magical. Now we can just compile cross-platform code and hand it to our friends.

Given that, I expect not to be using Python for these kinds of tools much going forward.


[glob]: https://doc.rust-lang.org/glob/glob/index.html
[clap]: https://clap.rs
[lld]: http://lld.llvm.org
[rustc-msvc]: https://github.com/rust-lang/rust/blob/master/src/librustc_trans/back/msvc/mod.rs#L300
[msvc]: http://landinghub.visualstudio.com/visual-cpp-build-tools

[^haskell]: Again: you can do similar with Haskell or OCaml or a number of other languages. And those are great options; they are in *some* ways easier than Rust—but Cargo is really magical for this.
