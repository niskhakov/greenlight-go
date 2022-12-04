## Creating handler (example: Update)

The nice thing about this handler is that we’ve already laid all the groundwork for it — our work here is mainly just a case of linking up the code and helper functions that we’ve already written to handle the request.

Specifically, we’ll need to:

- Extract the movie ID from the URL using the app.readIDParam() helper.
- Fetch the corresponding movie record from the database using the Get() method that we made in the previous chapter.
- Read the JSON request body containing the updated movie data into an input struct.
- Copy the data across from the input struct to the movie record.
- Check that the updated movie record is valid using the data.ValidateMovie() function.
- Call the Update() method to store the updated movie record in our database.
- Write the updated movie data in a JSON response using the app.writeJSON() helper.

```go
func (app *application) updateMovieHandler(w http.ResponseWriter, r *http.Request) {
    // Extract the movie ID from the URL.
    id, err := app.readIDParam(r)
    if err != nil {
        app.notFoundResponse(w, r)
        return
    }

    // Fetch the existing movie record from the database, sending a 404 Not Found 
    // response to the client if we couldn't find a matching record.
    movie, err := app.models.Movies.Get(id)
    if err != nil {
        switch {
        case errors.Is(err, data.ErrRecordNotFound):
            app.notFoundResponse(w, r)
        default:
            app.serverErrorResponse(w, r, err)
        }
        return
    }

    // Declare an input struct to hold the expected data from the client.
    var input struct {
        Title   string       `json:"title"`
        Year    int32        `json:"year"`
        Runtime data.Runtime `json:"runtime"`
        Genres  []string     `json:"genres"`
    }

    // Read the JSON request body data into the input struct.
    err = app.readJSON(w, r, &input)
    if err != nil {
        app.badRequestResponse(w, r, err)
        return
    }

    // Copy the values from the request body to the appropriate fields of the movie
    // record.
    movie.Title = input.Title
    movie.Year = input.Year
    movie.Runtime = input.Runtime
    movie.Genres = input.Genres

    // Validate the updated movie record, sending the client a 422 Unprocessable Entity
    // response if any checks fail.
    v := validator.New()

    if data.ValidateMovie(v, movie); !v.Valid() {
        app.failedValidationResponse(w, r, v.Errors)
        return
    }

    // Pass the updated movie record to our new Update() method.
    err = app.models.Movies.Update(movie)
    if err != nil {
        app.serverErrorResponse(w, r, err)
        return
    }

    // Write the updated movie record in a JSON response.
    err = app.writeJSON(w, http.StatusOK, envelope{"movie": movie}, nil)
    if err != nil {
        app.serverErrorResponse(w, r, err)
    }
}
```