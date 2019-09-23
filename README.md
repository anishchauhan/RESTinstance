# RESTinstance

[Robot Framework](http://robotframework.org) library for RESTful JSON APIs



## Advantages

1.  **RESTinstance relies on Robot Framework's language-agnostic, clean
    and minimal syntax, for API tests.** It is neither tied to any
    particular programming language nor development framework. Using
    RESTinstance requires little, if any, programming knowledge. It
    builts on long-term technologies with well established communities,
    such as HTTP, JSON (Schema), Swagger/OpenAPI and Robot Framework.
2.  **It validates JSON using JSON Schema, guiding you to write API
    tests to base on properties** rather than on specific values (e.g.
    "email must be valid" vs "email is foo@bar.com"). This approach
    reduces test maintenance when the values responded by the API are
    prone to change. Although values are not required, you can still
    test them whenever they make sense (e.g. GET response body from one
    endpoint, then POST some of its values to another endpoint and
    verify the results).
3.  **It generates JSON Schema for requests and responses automatically,
    and the schema gets more accurate by your tests.** Output the schema
    to a file and reuse it as expectations to test the other methods, as
    most of them respond similarly with only minor differences. Or
    extend the schema further to a full Swagger spec (version 2.0,
    OpenAPI 3.0 also planned), which RESTinstance can test requests and
    responses against. All this leads to reusability, getting great test
    coverage with minimum number of keystrokes and very clean tests.



## Installation

On 3.6, 3.7 you can install and upgrade
[from PyPi](https://pypi.org/project/RESTinstance):

    python3 -m venv venv
    source venv/bin/activate
    pip install --upgrade RESTinstance

On 2.7 series the package works as well, but using 2.7 is
[not preferred 2020 onwards](https://pythonclock.org/):

    virtualenv venv
    source venv/bin/activate
    pip install --upgrade RESTinstance

These also install [Robot Framework](https://pypi.org/project/robotframework)
if you do not have it already.



## Usage

There is a [step-by-step tutorial](https://github.com/asyrjasalo/RESTinstance/blob/master/examples) in the making, best accompanied with
[the keyword documentation](https://asyrjasalo.github.io/RESTinstance).

### Quick start

1.  Create two new (empty) directories `tests` and `results`.
2.  Create a new file `atest/YOURNAME.robot` with content:

``` robotframework
*** Settings ***
Library         REST    https://jsonplaceholder.typicode.com
Documentation   Test data can be read from variables and files.
...             Both JSON and Python type systems are supported for inputs.
...             Every request creates a so-called instance. Can be `Output`.
...             Most keywords are effective only for the last instance.
...             Initial schemas are autogenerated for request and response.
...             You can make them more detailed by using assertion keywords.
...             The assertion keywords correspond to the JSON types.
...             They take in either path to the property or a JSONPath query.
...             Using (enum) values in tests optional. Only type is required.
...             All the JSON Schema validation keywords are also supported.
...             Thus, there is no need to write any own validation logic.
...             Not a long path from schemas to full Swagger/OpenAPI specs.
...             The persistence of the created instances is the test suite.
...             Use keyword `Rest instances` to output the created instances.


*** Variables ***
${json}         { "id": 11, "name": "Gil Alexander" }
&{dict}         name=Julie Langford


*** Test Cases ***
GET an existing user, notice how the schema gets more accurate
    GET         /users/1                  # this creates a new instance
    Output schema   response body
    Object      response body             # values are fully optional
    Integer     response body id          1
    String      response body name        Leanne Graham
    [Teardown]  Output schema             # note the updated response schema

GET existing users, use JSONPath for very short but powerful queries
    GET         /users?_limit=5           # further assertions are to this
    Array       response body
    Integer     $[0].id                   1           # first id is 1
    String      $[0]..lat                 -37.3159    # any matching child
    Integer     $..id                     maximum=5   # multiple matches
    [Teardown]  Output  $[*].email        # outputs all emails as an array

POST with valid params to create a new user, can be output to a file
    POST        /users                    ${json}
    Integer     response status           201
    [Teardown]  Output  response body     ${OUTPUTDIR}/new_user.demo.json

PUT with valid params to update the existing user, values matter here
    PUT         /users/2                  { "isCoding": true }
    Boolean     response body isCoding    true
    PUT         /users/2                  { "sleep": null }
    Null        response body sleep
    PUT         /users/2                  { "pockets": "", "money": 0.02 }
    String      response body pockets     ${EMPTY}
    Number      response body money       0.02
    Missing     response body moving      # fails if property moving exists

PATCH with valid params, reusing response properties as a new payload
    &{res}=     GET   /users/3
    String      $.name                    Clementine Bauch
    PATCH       /users/4                  { "name": "${res.body['name']}" }
    String      $.name                    Clementine Bauch
    PATCH       /users/5                  ${dict}
    String      $.name                    ${dict.name}

DELETE the existing successfully, save the history of all requests
    DELETE      /users/6                  # status can be any of the below
    Integer     response status           200    202     204
    Rest instances  ${OUTPUTDIR}/all.demo.json  # all the instances so far
```

3.  Make JSON API testing great again:
```
robot --outputdir results atest/
```


## Contributing

Bug reports and feature requests are tracked in
[GitHub](https://github.com/asyrjasalo/RESTinstance/issues). We do respect pull request(er)s.

### Local development

We use [Nox](https://nox.thea.codes/en/stable/) over `make`, `invoke` and `tox`:
- Supports multiple Python versions, each session is ran on some `pythonX.X`.
- A session is a single virtualenv which is stored in `.venv/<session_name>`.
- Every `nox` recreates session, thus virtualenv, unless `reuse_venv=True`.

We test, develop, build and publish on Python 3.6, and use venvs as preferred:

    python3 -m venv .venv/dev
    source .venv/dev/bin/activate

Nox automates handling `.venv/<task>`s for workflows, that on Windows as well:

    pip install --user --upgrade nox

The actual tasks are defined in `noxfile.py`, as well as our settings like:
- The default Python interpreter to run all the defined tasks is `python3.6`
- We use [venv module](https://docs.python.org/3/library/venv.html) now for
virtualenving on Python 3
- Whether some virtualenv is always recreated when the particular task is ran (is our default)

Session is a task, running in the `.venv/<task>`, to list all possible sessions:

    nox -l

Default sessions are hilighted in the list, we run both `test`s and `atest`s by:

    nox

Session `nox -s atest` assumes you have started `testapi/` on [mountebank](https://www.mbtest.org):

    nox -s testenv

Running the above assumes you have `node` and `npx` installed in your system.

After started, you can debug requests and responses by tests in web browser at
[localhost:2525](http://localhost:2525/imposters).

You know, having a virtualenv even for generating libdoc - why not a bad idea:

    nox -s docs

Remove all sessions (`.venv/`s) as well as temporary files in your working copy:

    nox -s clean

Our distributions are known to work well on Python 3.7 and 2.7 series too:

    nox -s clean build

We use [zest.releaser](https://github.com/zestsoftware/zest.releaser) for
versioning, tagging and building wheels.

It uses [twine](https://pypi.org/project/twine/) underneath to upload to PyPIs
securely over HTTPS, which cannot be done with `python setup.py` commands.

This workflow is preferred for distributing a new (pre-)release to TestPyPI:

    nox -s test atest docs clean build release_testpypi install_testpypi

If that installed well, all will be fine to let the final release to PyPI:

    nox -s release

To install the latest release from PyPI, and do it in an own venv as usual:

    nox -s install

### pre-commit hooks

We want our static analysis checks ran before code even ends up in a commit.

Thus both `nox` and `nox -s test` commands bootstrap
[pre-commit](https://pre-commit.com/) hooks in your git working copy.

The actual hooks are configured in `.pre-commit-commit.yaml`.

### Ideas

- export `mb` recorded responses to CI (pre-commit hook: `nox -s save_testenv`)
- change `nox -s testenv` to load the saved testenv -> rid of `--allowInjection`
- add CI (GitHub Actions? GitLab?)
- add Python types to pass `prospector --with-tool mypy`
- enable pre-commit hook for prospector



## Credits

RESTinstance is under
[Apache License 2.0](https://github.com/asyrjasalo/RESTinstance/blob/master/LICENSE)
and was originally written by [Anssi Syrjäsalo](https://github.com/asyrjasalo).

It was first presented at the first [RoboCon](https://robocon.io), 2018.

Contributors:

  - [jjwong](https://github.com/jjwong) for helping with keyword
    documentation and examples (also check
    [RESTinstance_starter_project](https://github.com/jjwong/RESTinstance_starter_project))
  - [Przemysław "sqilz" Hendel](https://github.com/sqilz) for using and
    testing RESTinstance in early phase (also check
    [RESTinstance-wrapper](https://github.com/sqilz/RESTinstance-wrapper))
  - [Vinh "vinhntb" Nguyen](https://github.com/vinhntb),
    [#52](https://github.com/asyrjasalo/RESTinstance/pull/52).
  - [Stavros "stdedos" Ntentos](https://github.com/stdedos),
    [#75](https://github.com/asyrjasalo/RESTinstance/pull/75).
  - [Nicholas "bollwyvl" Bollweg](https://github.com/bollwyvl),
    [#84](https://github.com/asyrjasalo/RESTinstance/pull/84).

We use following Python excellence under the hood:

  - [Flex](https://github.com/pipermerriam/flex), by Piper Merriam, for
    Swagger 2.0 validation
  - [GenSON](https://github.com/wolverdude/GenSON), by Jon "wolverdude"
    Wolverton, for JSON Schema generator
  - [jsonpath-ng](https://github.com/h2non/jsonpath-ng), by Tomas
    Aparicio and Kenneth Knowles, for handling JSONPath queries
  - [jsonschema](https://github.com/Julian/jsonschema), by Julian
    Berman, for JSON Schema validator
  - [pygments](http://pygments.org), by Georg Brandl et al., for JSON
    syntax coloring, in terminal <span class="title-ref">Output</span>
  - [requests](https://github.com/requests/requests), by Kenneth Reitz
    et al., for making HTTP requests

See
[requirements.txt](https://github.com/asyrjasalo/RESTinstance/blob/master/requirements.txt)
for all the direct run time dependencies.

REST your mind, OSS got your back.
