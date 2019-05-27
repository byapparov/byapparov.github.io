# Data pipeline acceptance testing with examples

I have been looking for a way to introduce [Acceptance test driven development](https://en.wikipedia.org/wiki/Acceptance_test-driven_development) (ATDD) and [specification by example](https://en.wikipedia.org/wiki/Specification_by_example) into development of data pipelines in [MADE](https://www.made.com/). I needed something simpler compared to popular acceptance testing frameworks for Java. I also needed this for R projects as the first priority, where we already used mock data for testing. The biggest problem was that mock data is not visible to a user and hence not used to document system's behaviour. Instead code investigation with a developer required, which is time consuming and unreliable.

**Specification by example** is known to
bring improvement to development process and quality of a solution through:

* reduced communication errors between developers and  business users;
* better engagement of users in requirements gathering process;
* cross-team [single source of truth](https://en.wikipedia.org/wiki/Single_source_of_truth) documentation.

Putting data inputs and outputs into the spreadsheets that can be accessed by the user was a non brainer first option that we tried.

As user of the G Suite we trialed Google Spreadsheets and uptake from users is great. In one day we gathered meaningful feedback on the specification which allowed to build complete specification. Before it could take up to a month to get all formulas and conditions from the users in one place.

The next step was to organise examples in spread sheets to make them part of the integrated documentation, we used following rules:

* Name of the spreadsheet starts with the code of the story;
* There must be an automated way to update the mock data in the automated tests with examples from the spreadsheets;
* Spreadsheets should be referenced in the ticket.

I have introduced a new package
[usegs](https://github.com/byapparov/usegs) that integrates data from to make testing really simple for developers.

With a single call data can be added to mock data folder of your project.

```R
# install.packages("devtools")
devtools::install_github("byapparov/usegs")

## This code ran in the working directory of package
## will create files in `tests/testthat/data/{sheet-name}/` folder
use_gs_acceptance(
  x = "1bkoQYLYAVqgP4bCoqVe-yDB1mdX83cFtOqJ7q8GkT-w",
  extension = "csv"
)
```

Lets see what happens after `use_gs_acceptance` call.

Following files will be created for [example provided in the package](https://docs.google.com/spreadsheets/d/1bkoQYLYAVqgP4bCoqVe-yDB1mdX83cFtOqJ7q8GkT-w):

```
.
└─tests
  └──testthat
      │  test-usegs-acceptance.R
      └──data
           └──usegs-acceptance
                eta_etd_changed.csv		
                purchase_order_approved.csv
                file-out.csv			
                single-file-in.csv
                order_intake_complete.csv
                sud_updated.csv
```

Example is result of the following logic template:

```
.
└─tests
  └──testthat
      │  test-{acceptance name}.R
      └──data
           └──{acceptance name}
                {mock_file field value}.csv		
                {tab name}.csv			
```

*Note*: You can also change destination of your test data if you are not using `testthat`. See example with `.acceptance.yml` in [repository](https://github.com/byapparov/usegs)'s readme file.

*Note*: `mock_file` column in the spreadsheet can be used to save data into multiple mock files.

`test-*` file will contain blank test with exception to prompt that test's logic needs to be implemented.

Linking acceptance to the code with ability to override test mock files directly from examples that are shared with users should speed up test development and reduce the number of bugs.  
