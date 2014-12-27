Github Pull Request Status
==========================

At my current company, Captricity, we make heavy usage of github pull requests
to implement the practice of code reviews. However, oftentimes we run into
situations where a pull request languishes for a long time because we forget
what was the last action taken for it. All to often I hear the phrases "I was
waiting on you to fix it" or "I fixed it and was waiting for you to see it."
Part of the problem lies in the github pull request dashboard not providing all
the information necessary. I decided to write a script that will provide me a
dashboard that tells me which PRs need action from me.

A typical workflow for a pull request is as follows:

1. PR is opened
2. PR is reviewed and has comments that need addressing
3. All comments are addressed and is awaiting final review
4. Waiting for a clean build
5. Merge!

In each of these steps, it is clear what needs to happen to advance to the next
state:

1. Waiting for an initial review by assignee
2. Waiting for implementer to address feedback
3. Waiting for final review by assignee
4. Waiting for CI system
5. Done!

We want the dashboard to expose this information in a clear matter. Luckily, it
is fairly easy to determine what state the pull request is in:

1. No comments have been made by someone other than the requester
2. Last comment on the pull request is by the assignee
3. Last comment on the pull request is by the requester
4. A build has been triggered and is executing in CI OR the latest build has failed
5. PR is done!

The goal of the script is to use the above heuristic to determine what state
each of the open pull requests are in. To keep things simple, we will load the
full list of pull requests from github each time using the API to avoid
synchronization issues. After all, we're not building a hosted service. The
general flow of the application is as follow:

Grab list of open PRs -> Determine state of each PR -> Display in dashboard

You can view my code [here](https://github.com/yorinasub17/ghstat). The rest of
this post will be an overview of the code.

Accessing Github
----------------

I used the [PyGithub](http://jacquev6.net/PyGithub/v1/) library for interacting
with github, which provides a pythonic interface into the github v3 API. This
also provides nice classes for interacting with github objects that I can use
directly in the script. For example, getting a list of open pull requests for a
repository is as easy as:

```python
client = Github(API_KEY)
open_pulls = client.get_repo(REPOSITORY_NAME).get_pulls(state='open')
```

The api key is managed using the system's keyring service, which can be
accessed using the [keyring](https://pypi.python.org/pypi/keyring) library.
This provides for a more user friendly service by having the user only enter
the api token once and securely storing it in a manner that is retrievable.
The `get_api_key` routine handles the getting and setting of the api token:

```python
def get_api_key():
    api_key = keyring.get_password('ghstat', getpass.getuser())
    if not api_key:
        api_key = getpass.getpass('Enter Github API Key: ').strip()
        keyring.set_password('ghstat', getpass.getuser(), api_key)
    return api_key
```

Determining blockers
--------------------

We use the heuristic laid out above to determine blockers for pull request.
This is done in the function `determine_blockers`, which use the information on
the pull requestto determine its state:

```python
def determine_blockers(pull_request):
    # Get info about comments
    comments = sorted(pull_request.get_issue_comments(), key=operator.attrgetter('created_at'), reverse=True)
    if comments:
        last_comment = comments[0]
    else:
        last_comment = None
    commenters = {comm.user.login for comm in comments if comm.user.login != pull_request.user.login}

    # Now determine blockers based on workflow heuristics
    blockers = []
    if len(commenters) == 0:
        blockers.append((PullRequestBlocker.INITIAL_REVIEW, pull_request.assignee.login))
    elif last_comment and last_comment.user == pull_request.assignee:
        blockers.append((PullRequestBlocker.ADDRESS_FEEDBACK, pull_request.user.login))
    elif last_comment and last_comment.user == pull_request.user:
        blockers.append((PullRequestBlocker.FINAL_REVIEW, pull_request.assignee.login))

    if not pull_request.mergeable:
        blockers.append((PullRequestBlocker.MERGE_CONFLICTS, pull_request.user.login))
    return blockers
```

Note that we return a list of blockers. This is because a pull request could be
blocked on multiple issues, and we want to expose all of it in the dashboard.
We also return the person who needs to take action to unblock each particular
blocker. This could have been extracted separately, but I decided to return it
with the blocker for conciseness.

I used [enums](https://pypi.python.org/pypi/enum34) to represent each pull
request blocker. Enums are great when you have a canned set of values because
not only are they clear, but they also help catch bugs converting typos into
exceptions instead of silently failing. The enum is defined as follows:

```python
class PullRequestBlocker(Enum):
    INITIAL_REVIEW = 1
    ADDRESS_FEEDBACK = 2
    FINAL_REVIEW = 3
    CLEAN_BUILD = 4
    MERGE_CONFLICTS = 5
```

Displaying the data
-------------------

I used the [tabulate](https://pypi.python.org/pypi/tabulate) library for
displaying the data in a pretty table on the console. This library takes care
of all the formatting and spacing to ensure that the table renders correctly in
the console. It takes in a list of list representation of the table, so we need
to construct our table into a list of lists by iterating over the list of open
pull requests:

```python
data = []
for pull in open_pulls:
    blockers = determine_blockers(pull)
    users_needing_to_act = ', '.join(set(map(second, blockers)))
    blocker_codes = ', '.join(map(compose(str, first), blockers))
    data.append((pull.title, pull.user.login, pull.assignee.login, users_needing_to_act, blocker_codes))
print tabulate.tabulate(data, headers=headers)
```

If you are not familiar with functional programming, the functions `compose`,
`first`, and `second` might be foreign to you. They come from the library
[toolz](http://toolz.readthedocs.org/) which implements classic functional
programming constructs in python.


I hope this was useful in understanding the what, why, and how of the simple
ghstat script I wrote.
