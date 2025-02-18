- okay, it seems like this openat method might actually work
  - I've got most of the unit tests passing except for a few
  - seems like there are a few major types of issues in the unit tests:
    - some sort of issue with some tests involving symlinks
      - get an error about filesystem loop, too many links, something like that
    - errors simply arising from the fact that were not handling printing error messages properly and just panicking
      - this should be easy enough to fix
    - error about having too many files open
      - this should be fixed when we switch to `read_dir` being an actual iterator, so we only have one file open per level at a time

- next steps:
  - check if actually passes GNU test
  - fix broken tests:
    - switch `read_dir` to provide actual iterator, see if this fixes the failure for having too many files open
    - figure out why the symlink tests are failing
    - clean up all unwraps and whatnot, add actual error-handling
  - make whatever slight modifications are needed to the windows code path to make sure that's not broken
  - refactor to get actually decent performance
    - don't open files unless they are directories, otherwise just stat them (using `statat`) to get their metadata without opening

- okay, seems like separating stat and open (and only opening when we actually have a directory) is going to be essential to passing the symlink tests
  - this gets a bit tricky, because now we have to handle processing the stat result ourselves, instead of using the convenient `Metadata` data type
    - this gets kind of ugly, need to basically copy a lot of code from `cap-std` to see how they process the stat result
    - also a bit annoying because there is some platform-dependent variation,
      - for example, most Unix-like platforms have an `st_birthtime` field we can use, but for Linux I think we can only get this information from `Statx`, which requires a separate code path:
        - https://github.com/bytecodealliance/cap-std/blob/dcead54dba1ae9f519d2a7c0e91713549d44f3fa/cap-primitives/src/rustix/fs/stat_unchecked.rs#L33
        - probably also worth checking how Rust standard library implementation of `Metadata` handles this difference...

===============================
- okay, I think I've got a decent general plan for handling the rest of this:
  - switch `du` function to only take directories
  - in current layer of resursion, change directory to child just before calling `du` on child
    - then cd back to current layer as soon as call returns, then we can print an error here and return if we fail to cd back to main directory
    - hmm, this might not play nice with the plan to potentially only `cd` when absolutely necessary...
      - does `env::set_current_dir` also fail if path is too long?
        - TODO: test this!

- is only changing directories when directory path is too long a viable option?
  - what if directory path is just below limit, but then it has a child which is above the limit, does that cause a problem?
    - hmmm, it must not, otherwise the error that sparked this whole issue would have occurred when creating the `Stat` for the too-long directory, as opposed to causing a problem when trying to read it...

- this might also fix the bad handling when we fail to read directories, test that and maybe make a unit test for it

- still not totally sure how to handle the case where we can't get back up to the original directory...
  - would be nice if we could maybe return an error?
  - also still not sure if parent should be responsible for changing to and from child directory, or if the child should handle the directory change itself
    - ehh, I think both approaches end up being pretty similar, and it should be easy enough to change between the two...
  - might make sense to actually use a stack with (like pushd and popd)
    - but will these also have an issue with directory names that are too long?

- should add tests to handle when starting directory is something like `somedir/subdir/` and then we also pass other paths, to make sure we actually end up back at the original path at the end of the search

- next steps:
  - figure out if we can change directory to directory greater than path max
    - if not, need better solution for making sure we can go back up the directory stack eventually and be able to handle failures if we try to go back up the stack and fail to change directories
    - or, maybe just passing the GNU test is good enough, even if it's not super generalizable?
      - for example, maybe we can get a way with a simpler implementation if we only need to handle lengths over MAX_PATH but under 2*MAX_PATH?

- tenative plan:
  - have directory handle changing to and from itself as needed
    - only change directory if we get an error reading the directory because the path is too long
      - seems like we want to match on `e.kind() == std::io::ErrorKind::InvalidFilename`, but this error kind is not actually stabilized in the Rust stdlib yet...
        - relevant issues/PRs:
          - https://github.com/rust-lang/rust/issues/86442
          - https://github.com/rust-lang/rust/issues/130192
          - https://github.com/rust-lang/rust/pull/128316
          - https://github.com/rust-lang/rust/pull/134076
        - in the meantime, we can maybe match either on the string associated with the error kind ("invalid filename"), or just try changing directory if we get any error at all?

    - have `du` call return an error if we can't change to the directory, or we fail to change back to the parent
      - will need to update return type to return either `Box<dyn Error>` or a custom error enum created using `thiserror`
  - use `Path::parent` to cd back up to parent?
  - test if we can capture `read_dir` error specifically for path being too long
    - can we cd into the directory, or do we need to cd into its parent (since the directory's pathname is too long too `read_dir`, will that cause issues with cd as well?
  - initially implement just changing directory for every new directory, just as a PoC
    - once that's working, update to only change directory when absolutely necessary to path being too long

- can `Path::parent` return `".."` ?

- tests to possibly add:
  - test that output upon failure to access directory actually matches GNU behavior
  - should add tests to handle when starting directory is something like `somedir/subdir/` and then we also pass other paths, to make sure we actually end up back at the original path at the end of the search
  - do we have tests to cover using `"."` vs. "" for input path?
    - and for `"./somedir"` vs. `"somedir"`
  - test that when we use a mix of absolute and relative paths, it actually works...
    - maybe separate tests for absolute path target where absolute path of starting directory is within limit and one where exceeds limit

- ahh crap, this might get funky if we have an input that mixes absolute and relative paths?
  - if we do absolute path first, what directory do we end up in?
    - wouldn't this cause a problem because we go back up to root directory then call `cd ..`?
  - if we're using absolute paths, I think we'd have to get the absolute path of the directory we start in, then change directories back to it after we finish searching the absolute path we started in
    - we'd have to either assume the absolute path of the directory we start in is within path max, or cd into incrementally starting from the root?
    - to minimize potential issues, we should avoid changing directories during dfs unless we absolutely have to, so then ideally we just won't need to change back to our starting directory

- next steps:
  - test if `set_current_dir` works with directories exceeding max path length
    - nope, it errors out
  - remove unwraps, add proper error handling
    - okay, got this done, might want to clean it up a bit later though
  - add test to ensure we end up back in right place if use an absolute path followed by a relative path
    - maybe the test should explicitly confirm we end up back in the original directory?
  - add logic to ensure we end up back in the directory we started with even when using an absolute path
  - add tests for directories exceeding max path length?
    - one for relative path, one for absolute path
  - add test where first target path has multiple components, followed by a path with only one component
  - add test for path that starts with some amount of `".."` (parent directory) component(s)
  - add test to ensure output when we have an inaccessible directory actually matches GNU output
    - seems like this already exists
  - add logic to only change directories when absolutely needed due to file path being too long
    - ask in Discord how to handle the fact that the InvalidFilename error kind isn't stable yet
  - add tests to cover absolute paths
    - used `env::current_dir` to get absolute path to use as input
 
- ahhh, we have another potential snag: what if the user provides a relative path but includes `".."` (parent directory) components?
  - my idea of just changing directories to parent directories to go back up the stack won't work in this case
  - ooh okay, I think I have an idea:
    - have stack of pathbuf objects containing the absolute path, put a new element in the stack every time we hit an error for the path being too long
        - and we can maybe use a similar mechanism for handling absolute paths and returning back to starting directory

- honestly the `du` function should probably be turned into a struct with some methods and whatnot, that would make this stuff a lot cleaner
  - yeah honestly a refactor into a proper class is sorely needed...
    - maybe I can handle that at some point...

- I might just want to drop a note in the issue or in the discord that it seems like we can't really do this until that particular error is stabilized
  - and then in the meantime I can maybe keep working on this using nightly rust or something?

- okay, based on comments in the `du/long-from-unreachable.sh` test, seems like GNU uses `openat` to be able to handle long paths without actually needing to change directories
  - looks like there is a Rust library for this but it doesn't seem particularly robust
    - actually, it's entirely unmaintained
    - looks like there is another crate called `Rustix` which has this and seems more actively maintained which has an `openat` implemenation which seems like it could work
      - still, I think that only works for Linux
      - but it looks like Windows has an `NtCreateFile` API function that can do something similar?
    - seems like the `cap-std` crate might have a cross-platform implemenation we can use?
      - https://github.com/bytecodealliance/cap-std/blob/dcead54dba1ae9f519d2a7c0e91713549d44f3fa/cap-std/src/fs/dir.rs#L290

- okay, seems like there are primarily two directions I can go down here:
  - actually change current directory in order to shorten relative paths
    - for performance reasons, we probably only want to change directories when we hit a filename that is too long and causes an error
      - requires `std::io::ErrorKind::InvalidFilename`, which is not yet stabilized, in order to detect this specific failure
    - in order to pass GNU test, once we get an input with an absolute path, we will want to avoid trying to return to original directory unless we have later inputs which are relative paths
      - otherwise we are fine to just keep using absolute paths and never returning to original directory
  - don't actually change current directory, just use something like `openat` to be able to open directories using their path relative to some other already-open directory
    - this seems preferable in theory, seems to have less opportunity for unintended consequences
    - it seems like the most robust cross-platform implementation of this type of functionality is in `cap-std`
    - probably can just use this at every level of recursion, thus no need to specifically catch `std::io::ErrorKind::InvalidFilename`
    - I need to test this functionality out to see if it actually works the way I expect
    - also need to make sure it actually works on all platforms we need
    - also need to be careful that we can actually use the `cap-std` stuff without breaking `struct Stat`

- either way, we will probably want to refactor `du` into an struct with some methods (most of which should be inlined) to make adding these changes cleaner and more straightforward, and just generally make maintenance easier

- next steps:
  - do some toy tests with `cap-std` to see if it works the way I think it does
    - looks good
  - confirm that `cap-std` actually supports all of the platforms we need
    - seems like it covers Linux/windows/mac, so probably fine?
  - make post in issue describing options I'm currently looking at
  - if `cap-std` is actually a viable option based on my tests, create a fork of my current branch where I try to actually use it
  - add unit tests to cover long paths:
    - as relative path (+ with some second path argument
    - as absolute path (+ with some second path argument)
  - if `cap-std` stuff didn't work out, try passing tests using the directory-changing approach

- okay, the cap-std stuff seems to mostly work, with a few concerns that may or may not be show-stoppers:
  - seems like `Dir::metadata` throws an error if symlink leads out of the filesystem
    - specifically a custom `ErrorKind::PermissionDenied`
      - I should see where this factors in to see if I can circumvent it somehow...
    - which makes sense, since one of the purposes of this `cap_std` stuff is to provide sandboxes for WASM stuff
    - is there a way around this?
      - maybe by using one of the lower-level APIs, which might be able to gives us the cross-platform `openat`-style functionality without adding the sandboxing stuff on top of it
      - okay, looks like theres `read_link_unchecked` function, which isn't public, but:
        - for Unix-like, it just calls `rustix::fs::readlinkat`, which I think is a public thing we can maybe try using?
          - need to see how `Dir::metadata` actually works to see how we should use the result of following the link
        - for Windows, it just calls `fs::read_link` with the full path
          - are file path length limits not a thing anymore on Windows? I think I read something indiciating that might be the case at some point, but I'm not sure if it's actually true
            - I suppose it's something I could test
  - the `get_file_on_disk` and `get_file_info` functions for windows currently take full paths
    - how hard would it be to update these to work with this new paradigm, perhaps using a file + relative path?
  - I should also at some point assess what the overall performance impact is of this...

TODO:
- clean up unwraps and panics
  - proper error handling if we fail to change directories

- run some sort of performance benchmarks?

- confirm GNU test is now passing
  - and confirm we didn't break any existing GNU tests
    - how can we do this locally?
  - failing to build GNU tests (I think due to lack of SELinux headers in WSL2 kernel)
    - tried enabling SELinux, need to reboot PC and try compiling again to see if it worked
      - if this doesn't work, should try doing all of this on my old Linux machine, as the SELinux stuff shouldn't pose a problem there
    - is there some other way to run the GNU tests? they're just bash scripts so I feel like it shouldn't be that hard...

  - okay, made a copy of the test that doesn't rely on building all the GNU stuff, seems like my new code is passing the tests
    - now just need to do the rest of my TODO list...

- is there a cleaner way to create `full_path` variable?

- handle moving up and down directories more cleanly
  - maybe use some sort of RAII to make sure we go back up directory upon exiting the scope of the function?

- maybe only change directories when we actually hit an error with the path being too long, instead of just always doing it for every recursive call

- rebase and/or squash down to one commit
  - make separate branch before trying this, so we still have all of our code even if we mess up the git surgery

- add unit tests?

- run clippy and rustfmt

FUTURE ISSUE?
- fix error handling when we can't access directory
  - don't print bad info for the directory, just print the failure
    - will probably need to have directories print their stats after calling `du` on all of their children
      - as opposed to current arrangement where have the parent doing the print, which is frankly just kind of dumb

  - write some tests to show this problem, submit issue
    - add unit tests to prevent regression
