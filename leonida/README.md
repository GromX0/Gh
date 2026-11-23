# Leonida integration branch (`boot-chain`)

`boot-chain.bundle` holds our work on top of upstream Leonida `main` `042c5f37`: one test fix and the open upstream
pull requests for the retail boot chain, merged in boot-chain order. Conflict resolutions are explained in each merge
commit. The list of what is included is in `../PROGRESS.md` ("Integration branch").

No game data is in here: only source code from the public repository and our merges.

## Restore it (any machine with git)

```sh
git clone https://github.com/ItsNotRexed/Leonida leonida
cd leonida
git fetch origin 042c5f37836dc2c8d5913c2fc5a88e99bdee083b   # the bundle's base commit
git bundle verify /path/to/boot-chain.bundle
git fetch /path/to/boot-chain.bundle boot-chain:boot-chain
git checkout boot-chain
git submodule update --init                                     # third_party/rpf9-explorer, for the asset stage
```

Then build as the upstream README says (`Build.bat`, or `python -I make_game.py ...`). The `runtime` stage builds
`gta6rt` from this checkout.
