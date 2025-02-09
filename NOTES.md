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
 
- ahhh, we have another potential snag: what if the user provides a relative path but includes `".."` (parent directory) components?
  - my idea of just changing directories to parent directories to go back up the stack won't work in this case
  - ooh okay, I think I have an idea:
    - have stack of pathbuf objects containing the absolute path, put a new element in the stack every time we hit an error for the path being too long
        - and we can maybe use a similar mechanism for handling absolute paths and returning back to starting directory

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
