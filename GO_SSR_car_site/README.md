## Go SSR-only / std packages only cars stats website

Using a limited set of cars provide in a test API. The main goal was to smooth out the missing asynchronous front-end layer as much as possible - which would be used under normal constraints.
*No code in repo, as pushing code itself to public repos is not allowed by school policies.

### Click-through video:

[![Click-through video](https://img.youtube.com/vi/db6RcIiLt7k/0.jpg)](https://www.youtube.com/watch?v=db6RcIiLt7k)


### Usage
1. Run the build command in path/to/cars/backend/api (might need to reinstall node.js if one is present in the system):

	```
	make build	
	```

2. Start API with the following command in path/to/cars/backend/api:

    ```
    make run
    ```
3. Start server in root wih the following command"

    ```
    go run server.go
    ```
4. Page runs on [localhost](http://localhost:8000/).

### Things not done:
- Footer does not have any actual links to pages atm, as no real content to show there - Footer is added purely for the complete looks.

- Mobile responsiveness - was not required, so omitted this atm. Otherwise would have made more sense to start with mobile-first design vs scaling down from data-rich desktop layout.

- With 10 cars only in the API, did not paginate the home page with all cars - IRL this would be paginated to lazy load a subset of cars and hide rest in "next pages"

### Scope:

- Did not use non-std Go packages, although it seems that gorillamux (not in std pakage) is actually the default way to handle url routing. Thus there are some weird net/http URL routing workarounds because of the need to get data from URL paths and car search URL query values.

- Did use JS for 1 thing for handling extras - when the comparison bar has 2 cars (which is the maximum I allowed), then JS stops from user adding more cars as the cookie does not accept more before removing a car - this would IMO be front-end validated anyway IRL, because user should get instant feedback. Thought about doing that with htmx, but with so much data being pased around between handlers and funcs this would have been more error-prone and JS seems more logical for this anyway.

### General logic:

- As the goal was to use Server-Side Rendering (SSR) and no full frontend asynchronous support, tried to achieve as close of a UX feel for a full frontend framework or AJAX usage.

- As SSR does come with the issue of full reloads, I used cookies and helper funcs to include data for eaxh template so that when user does something which should not redirect them, they will "remain in the same page" with the same state of the page. Meaning that, while page is reloaded, the page is built back the same way.

- Used a channel with goroutines to run the serach for cars in teh sidebar search. With 10 cars in the API, it is an overkill, but a good place to use channels and goroutines if this would be a real life car sales page, etc.

- Interestingly, as searchbar has 8 "decoupled" search categories, there was an opportunity to track how many searches actually run in the channel and then compare the count of searches vs some care appearing in the searches. If the search count and result count for some car matches, then the car is the one we are looking for - kind of an anonymous search where context does not matter. Would not probably scale really well, but kept that logic, as it just is interestng and works.



    ```Go

    eightResults := make(chan []models.Car, 8)
	// keep track of number of same0time searches - can use to "find" cars too
	activeSearches := 0

	// running independent seacrhes at the same time with form values from above:
	if manufacturer != "" {
		go services.FilterCars("manufacturer", manufacturer, carsSidebar, eightResults)
		activeSearches++
	}
	if category != "" {
		go services.FilterCars("category", category, carsSidebar, eightResults)
		activeSearches++
	}
	if transmission != "" {
		go services.FilterCars("transmission", transmission, carsSidebar, eightResults)
		activeSearches++
	}
	if model_year != "" {
		go services.FilterCars("model_year", model_year, carsSidebar, eightResults)
		activeSearches++
	}
	if drivetrain != "" {
		go services.FilterCars("drivetrain", drivetrain, carsSidebar, eightResults)
		activeSearches++
	}
	if engine != "" {
		go services.FilterCars("engine", engine, carsSidebar, eightResults)
		activeSearches++
	}
	if horsepower != "" {
		go services.FilterCars("horsepower", horsepower, carsSidebar, eightResults)
		activeSearches++
	}
	if country != "" {
		go services.FilterCars("country", country, carsSidebar, eightResults)
		activeSearches++
	}

	// append found Car structs into a slice
	var Cars []models.Car
	for i := 0; i < activeSearches; i++ {
		result := <-eightResults
		Cars = append(Cars, result...)
	}

	// as searches are independent, can compare searches ran nr to duplicate matches found - if no match, no valid car found, if match, this/hee are the correct cars - kind of hacky but works with this limited API and no async in frontend
	appearanceCount := make(map[int]int)
	for _, car := range Cars {
		appearanceCount[car.ID]++
	}

	// if count mactehs searches run - then our car to be sent as data
	var filteredCars []models.Car
	var wasFound bool = false        // to trigger msg data or car data
	uniqueCars := make(map[int]bool) // using boolmap workaround to get unique cars only into the strucstslice
	for _, car := range Cars {
		if appearanceCount[car.ID] == activeSearches {
			if _, exists := uniqueCars[car.ID]; !exists {
				wasFound = true
				filteredCars = append(filteredCars, car)
				uniqueCars[car.ID] = true // marking cars when adding, so duplicates are not added
			}

		}
	}

    ```

- IRL would likely optimize searches with more custom API endpoints, but here used the API "as is", thus loaded all cars to a model and used the model when and where needed.
- For search sidebar feature needed to use a copy of the car model with 1 extra field, due to SSR. To preserve user's search results with redirects and due to cars being renderd in templates with range codeblock, needed to access current url query in the car model range for the "add to comparison" button" I just added the query return value {{.ReturnPath}} to the copy of the car model, so it would be accessible.

    ```html

    {{range .Cars}}
    <div class="one-car">


        <div>
            <a href="/cars/{{.URLname}}/{{.ID}}">
                <img src="/api/img/{{.Image}}" alt="{{.Name}}">
            </a>
        </div>

        <div class="text-box">

            <div class="homecar-text">
                <h3>{{.Name}}</h3>
                <p>{{.Year}}</p>
                <p>{{.Specifications.Transmission}}</p>
                <p class="country">{{.ManufacturerCountry}}</p>
            </div>

            <div class="homecar-buttons">
                <div class="car-details">
                    <a href="/cars/{{.URLname}}/{{.ID}}">
                        <img class="expansion" src="/public/img/chartbar.svg" alt="Car specifications and details">
                    </a>
                </div>
                <div class="add-comparison">
                    <a href="/add-comp/{{.ID}}/?returnUrl=/results/?{{.ReturnPath}}">
                        <img class="comparison" src="/public/img/compare.svg" alt="Add the car to comparison">
                    </a>
                </div>
            </div>

        </div>

    </div>
    {{end}}

    ```

### CSS/Design/UX logic:

- While it was a car stats page, built it as it would be a car sales/dealership platform - meaning that tried to keep the design utilitarian and with more of a desktop data-heavy look. Because car slaes pages kind of have a unified data-rich boxy style and goning very modern and minimalist here could mean an "uncanny valley" feeling for IRL users.

- Mixed grid and flex layouts and used svg heroicons - to reduce text-basd buttons and actionable links, as page is text/data-heavy anyway by default.

- To have a more balanced color pallette, used a gradient to darken out the page from utility-based sidebar on the left towards main content on the right.

- Kept text styles minimal and deliberately used std balck font for actual car previews and car details - IRL thes would or could be the main sales items or added by platform users, so these have to pop out more and not be smootly hidden among the general page design. To balanc this, did lighten the comparison page a lot by text and table design.

- For the car preview grid, I used placeholder divs to enforce and make the grid really robust. IRL and with more cars it would also need paggination to only load a certain number of cars, but here the placeholder divs make sure that the grid columns do not start taking up more space when only 1 row worth of cars is found (less than 4 cars). Adding 6 placeholder divs means that there is always at least 3 rows of "results" which maintains the grid well (1 car found + 6  divs = 3 columns and 3 rows | 3 cars found + 6 divs also = 3 columns and 3 rows).

    ```html

    {{if lt (len .Cars) 4}}
                        
        <div class="grid-placeholder"></div>
        <div class="grid-placeholder"></div>
        <div class="grid-placeholder"></div>
        <div class="grid-placeholder"></div>
        <div class="grid-placeholder"></div>
        <div class="grid-placeholder"></div>
        
    {{end}}

    ```

- Design + code logic: Did not deliberately stop a user from comparing a car vs itself or not running compare action or opening the compare page with 1 car only - User can compare a car vs itself and also open the comparison page with no car or 1 car only chosen - This is s because this edge case does not brake anything and if a user wants to be weird, there is no reason to spend time and resources to stop them from doing so IRL if it does not affect the business logic of the project.