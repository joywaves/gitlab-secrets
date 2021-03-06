# GitLab Secrets

GitLab offers a way to
[prevent pushing files with secrets](https://docs.gitlab.com/ee/push_rules/push_rules.html#prevent-pushing-secrets-to-the-repository)
that have a specific filename, but this doesn't go far enough because it
doesn't scan files for secrets.

AWS's git-secrets (https://github.com/awslabs/git-secrets) provides a
way to scan files but it requires users to install it on their
local machine.  This is great but it is hard to ensure users are
compliant with best practices.

A hybrid approach would be to have git-secrets on both client-side and
GitLab server-side checking for secrets. GitLab can reject a push if it
fails a scan, and users can use the same git-secrets tool to resolve
the problem from client-side.

## Demonstration

Demonstrate the use of git-secrets, with aws rules,
(https://github.com/awslabs/git-secrets) in a GitLab server-side
update hook.

### GitLab

Extends gitlab Docker image to include git-secrets, with aws rules, in
a custom update hook.

Start `gitlab` container in background:

    docker-compose up -d gitlab


Tail the logs to make sure it came up properly:

    docker-compose logs -f gitlab

Or watch the state for health status of `healthy`:

    watch docker-compose ps

Open a browser and navigate to http://localhost:8081 and initialize the
`root` password. 

For this demo we will use the following as the password:

    Welcome1

Once logged in, create a project under root group called:

    my-repo

### Git

A minimal Docker Alpine container with git and
[git-crypt](https://github.com/AGWA/git-crypt)
installed to test git-secrets.

Start `git` container:

    docker-compose run git /bin/sh

From within `git` container; clone `my-repo` project using `root/Welcome1` credentials:

    git clone http://gitlab:8081/root/my-repo.git

Change to `my-repo` directory:

    cd my-repo

#### Push non-secret and aws example keys

Verify you are able to push a non-secret change:

    echo "no secrets here" > README.md
    git add README.md
    git commit -m "Update readme"
    git push -u origin master


Verify you are able to push the default allowed AWS example key:

    echo "AKIAIOSFODNN7EXAMPLE wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY" > README.md
    git add README.md
    git commit -m "Allowed example secret"
    git push -u origin master


#### Override false positive

Verify you are NOT able to push an AWS secret key:

    echo "AKIAIOSFODSECRETSKEY wJalrXUtnFEMI/K7MDENG/bPxRfiCYSECRETSKEY" > README.md
    git add README.md
    git commit -m "Oops"
    git push -u origin master

Verify you are able to override GitLab blocking a push with
`.gitallowed` file:

    echo "SECRETSKEY" > .gitallowed
    git add .gitallowed
    git commit -m "False positive"
    git push -u origin master

#### Revert committed secret

Verify you are NOT able to push another AWS secret key:

    echo "AKIAIOSFODVERYSECRET wJalrXUtnFEMI/K7MDENG/bPxRfiCYVERYSECRET" > README.md
    git add README.md
    git commit -m "Oops"
    git push -u origin master

Backout the commit:

    git reset HEAD~1
    git checkout README.md

#### Git-crypt encrypted files allowed

Verify you are able to push git-crypt encrypted files:

    git-crypt init
    echo "my-repo-crypt.key" > .gitignore
    git-crypt export-key my-repo-crypt.key
    echo "my.secrets filter=git-crypt diff=git-crypt" > .gitattributes
    echo "password=muMj8RUUdJtzGazK" > my.secrets
    git add .
    git commit -m "git-crypt example"
    git push -u origin master

---

Note that `.gitallowed` works by adding secrets allowed to the git repo
config file for the project on the gitlab server. It would require
manual removal of a secret allowed, if that secret allowed was removed
from the `.gitallowed` file.
