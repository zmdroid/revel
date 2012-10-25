---
title: Booking
layout: samples
---

The Booking sample app demonstrates:

* Using a SQL database
* Interceptors for checking that the user is logged in.
* Using validation to display inline errors

Here are the contents of the app:

	booking/app/
		models		   # Structs and validation.
			booking.go
			hotel.go
			user.go

		controllers
			app.go     # "Login" and "Register new user" pages
			hotels.go  # Hotel searching and booking
			db.go      # A plugin for managing an in-memory SQLite

		views
			...


[Browse the code on Github](https://github.com/robfig/revel/tree/master/samples/booking)

## Installation

This app uses [go-sqlite3](https://github.com/mattn/go-sqlite3), which wraps the
C library.  To install (OSX):

1. Install Homebrew ( http://mxcl.github.com/homebrew/ ):
>	`ruby <(curl -fsSkL raw.github.com/mxcl/homebrew/go)`

2. Install pkg-config and sqlite3:
>	`brew install pkgconfig sqlite3`

Once you have SQLite installed, it should be possible to run the booking app as
usual:

	$ revel run github.com/robfig/revel/samples/booking

## Database Plugin

**app/controllers/db.go** defines `DbPlugin`, which is a plugin that does a couple things:

* OnAppStart: Opens a SQLite in-memory database, creates the User, Booking, and
  Hotel tables, and inserts some test records.
* BeforeRequest: Begins a transaction and stores the Transaction on the Controller
* AfterRequest: Commits the transaction.  Panics if there was an error.
* OnException: Rolls back the transaction.

[Check out the code](https://github.com/robfig/revel/blob/master/samples/booking/app/controllers/db.go)

## Interceptors

**app/controllers/hotels.go** registers an interceptor that runs before every
action on that controller:

<pre class="prettyprint lang-go">
func init() {
	rev.InterceptFunc(checkUser, rev.BEFORE, &Hotels{})
}
</pre>

`checkUser` looks up the username in the session and redirects the user to log
in if they are not already.

<pre class="prettyprint lang-go">
func checkUser(c *rev.Controller) rev.Result {
	if user := connected(c); user == nil {
		c.Flash.Error("Please log in first")
		return c.Redirect(Application.Index)
	}
	return nil
}
</pre>

[Check out the user management code in app.go](https://github.com/robfig/revel/blob/master/samples/booking/app/controllers/app.go)

## Validation

The booking app does quite a bit of validation.

For example, here is the routine to validate a booking, from
[models/booking.go](https://github.com/robfig/revel/blob/master/samples/booking/app/models/booking.go):

<pre class="prettyprint lang-go">
func (booking Booking) Validate(v *rev.Validation) {
	v.Required(booking.User)
	v.Required(booking.Hotel)
	v.Required(booking.CheckInDate)
	v.Required(booking.CheckOutDate)

	v.Match(b.CardNumber, regexp.MustCompile(`\d{16}`)).
		Message("Credit card number must be numeric and 16 digits")

	v.Check(booking.NameOnCard,
		rev.Required{},
		rev.MinSize{3},
		rev.MaxSize{70},
	)
}
</pre>

Revel applies the validation and records errors using the name of the
validated variable (unless overridden).  For example, `booking.CheckInDate` is
required; if it evaluates to the zero date, Revel stores a `ValidationError` in
the validation context under the key "booking.CheckInDate".

Subsequently, the
[Hotels/Book.html](https://github.com/robfig/revel/blob/master/samples/booking/app/views/Hotels/Book.html)
template can easily access them using the **field** helper:

<pre class="prettyprint lang-go">{% capture tmpl %}{% literal %}
  {{with $field := field "booking.CheckInDate" .}}
    <p class="{{$field.ErrorClass}}">
      <strong>Check In Date:</strong>
      <input type="text" size="10" name="{{$field.Name}}" class="datepicker" value="{{$field.Value}}">
      * <span class="error">{{$field.Error}}</span>
    </p>
  {{end}}
{% endliteral %}{% endcapture %}{{ tmpl|escape }}</pre>

The **field** template helper looks for errors in the validation context, using
the field name as the key.