# Instructions to using `fbs pro`

Here is a quick guide on how to use
GitHub Actions with `fbs pro`.
Details on `fbs pro`
can be found
[here](https://build-system.fman.io/pro).

Once you purchased `fbs pro`,
you will get a download link that can be used
to download the package via `pip`.
You probably do not want to add this URL
to your GitHub workflow file
or to the `requirements.txt` file, 
since this would make it public. 
However,
you can use 
[GitHub Secrets](https://docs.github.com/en/codespaces/managing-your-codespaces/managing-encrypted-secrets-for-your-codespaces)
in order to manage the download link securely.

*Note*: Here we will use a
repository secret to manage the download link
for `fbs pro`.
If you want to use an organization-wide secret,
you should be easily able to adopt
the instructions here.

## Setting up your GitHub Secret

To set up a repository secret,
go to your repository on GitHub 
and click Settings and navigate to Secrets.
You can now set up a new secret
as shown in the following screenshot.
![Add Secret](img/add_secret.png)
Then click `Add secret` and your download link
will safely be stored in the GitHub profile.
You can not retrieve the secret link anymore,
however,
you will be able to change its value later if required.


## Adjusting the `requirements.txt`

Next you want to adjust your `requirements.txt` file.
You do not want to install `fbs` anymore from PyPi directly.
Therefore, remove `fbs` from your `requirements.txt` file.

If you want to leave the file as is for users to install directly,
you can always create a second file and name it,
e.g., `requirements-gha.txt`. 
Then modify your action to install from this file
by stating:
```
pip install -r requirements-gha.txt
```

*Note*: If you modify the requirements file
you probably want to add a note for users that want
to compile the software by themselves.
If it requires `fbs pro`,
your user should have it installed as well.
Maybe make a note in a README file
or use something similar.


## Modifying the workflow file

With the modified `requirements.txt` file, 
we now also want to modify the workflow. 
In the task `Install dependencies`,
we so far upgrade `pip`
and install the `requirements.txt` file via `pip`.
You can modify this block as following:

```
python -m pip install --upgrade pip
pip install ${{ secrets.FBS_PRO_DL }}
pip install -r requirements.txt
```

The middle line here is where `fbs pro`
is installed. 
Replace `FBS_PRO_DL` with the variable name
of your repository secret.
From now on,
`fbs pro` will be installed prior 
to installing any dependencies.
Your download link is securely stored within the repository.
