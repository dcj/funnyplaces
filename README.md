# About

Funnyplaces is a Clojure client for Factual's API. It supports rich queries across Factual's U.S. Places, Crosswalk, and Crossref, and Resolve services.

Factual's [web-based API](http://developer.factual.com) offers:

* Places API: Rich queries across a high quality dataset of U.S. Points of Interest and business entities.
* Crosswalk: Translation between Factual IDs, third party IDs, and URLs that represent the same entity across the internet.
* Crossref: Lets you find URLs that contain entities in the Factual places database, or to find the Factual ID of a place mentioned on a URL.
* Resolve: An entity resolution API that makes partial records complete, matches one entity against another, and assists in de-duping and normalizing datasets.

# Installation

Funnyplaces is hosted at [Clojars](http://clojars.org/funnyplaces). Just add this to your dependencies:

	[funnyplaces "1.1-beta"]

# Basics

## Setup

	(ns yournamespace.core
	  (:use [funnyplaces.api :as fun]))
	(fun/factual! "YOUR_FACTUAL_KEY" "YOUR_FACTUAL_SECRET")

## Simple Fetch

	;; Fetch 3 random Places from Factual
	(fun/fetch :places :limit 3)

<tt>fetch</tt> takes the table name as the first argument, followed by a list of option pairs. It returns a sequence of records, where each record is a hashmap representing a row of results. So our results from above will look like:

	[{:status 1, :country US, :longitude -94.819339, :name Lorillard Tobacco Co., :postcode 66218, ... }
	 {:status 1, :country US, :longitude -118.300024, :name Imediahouse, :postcode 90005, ... }
	 {:status 1, :country US, :longitude -118.03132, :name El Monte Wholesale Meats, :postcode 91733, ... }]

This means it's easy to compose concise queries. For example:

	;; Get the names of 3 Places from Factual
	> (map :name (fun/fetch :places :limit 3))
	("Lorillard Tobacco Co." "Imediahouse" "El Monte Wholesale Meats")

## More Fetch Examples

	;; Return rows where region equals "CA"
	(fun/fetch :places :filters {"region" "CA"})

	;; Return rows where name begins with "Starbucks" and return both the data and a total count of the matched rows:
	(fun/fetch :places :filters {:name {:$bw "Starbucks"}} :include_count true)

	;; Do a full text search for rows that contain "Starbucks" or "Santa Monica"
	(fun/fetch :places :q "Starbucks,Santa Monica")

	;; Do a full text search for rows that contain "Starbucks" or "Santa Monica" and return rows 20-40
	(fun/fetch :places :q "Starbucks,Santa Monica" :offset 20 :limit 20)

# Places Usage

	;; Return rows with a name equal to "Stand" within 5000 meters of the specified lat/lng
	(fun/fetch :places
	       :filters {:name "Stand"}
	       :geo {:$circle {:$center [34.06018, -118.41835] :$meters 5000}})

# Crosswalk Usage

	;; Return all Crosswalk data for the place identified by the specified Factual ID
	(fun/crosswalk :factual_id "97598010-433f-4946-8fd5-4a6dd1639d77")

	;; Return Loopt.com Crosswalk data for the place identified by the specified Factual ID
	(fun/crosswalk :factual_id "97598010-433f-4946-8fd5-4a6dd1639d77" :only "loopt")

	;; Return all Crosswalk data for the place identified by the specified Foursquare ID
	(fun/crosswalk :namespace "foursquare" :namespace_id "4ae4df6df964a520019f21e3")

	;; Return the Yelp.com Crosswalk data for the place identified by a Foursquare ID: 
	(fun/crosswalk :namespace "foursquare" :namespace_id "4ae4df6df964a520019f21e3" :only "yelp")

# Crossref Usage

The <tt>get-factid</tt> function takes a URL and returns the Factual ID for the place mentioned on the specified URL. For example: 

	> (fun/get-factid "https://foursquare.com/venue/215159")
	[{:url "https://foursquare.com/venue/215159", :is_canonical true, :factual_id "97598010-433f-4946-8fd5-4a6dd1639d77"}]

The <tt>get-urls</tt> function takes a Factual ID and returns URLs that mention that place. For example:

	;; Return all URLs where the place identified by the specified Factual ID is mentioned
	(fun/get-urls "97598010-433f-4946-8fd5-4a6dd1639d77")

# Resolve Usage

The <tt>resolve</tt> function takes a hash-map of values indicating what you know about a place. It returns the set of potential matches, including a similarity score.

	> (fun/resolve {"name" "ino", "latitude" 40.73, "longitude" -74.01}))
	
The <tt>resolved</tt> function takes a hash-map of values indicating what you know about a place. It returns either a certain match, or nil.

	> (fun/resolved {"name" "ino", "latitude" 40.73, "longitude" -74.01}))

# Results Metadata

Factual's API returns more than just results rows. It also returns various metadata about the results. You can access this metadata by using Clojure's <tt>meta</tt> function on your results. Examples:

	> (meta (fun/fetch :places :filters {:name {:$bw "Starbucks"}} :include_count true))
	{:total_row_count 8751, :included_rows 20, :version 3, :status "ok"}

	> (meta (fun/crosswalk :factual_id "97598010-433f-4946-8fd5-4a6dd1639d77"))
	{:total_row_count 13, :included_rows 13, :version 3, :status "ok"}

	> (meta (fun/get-factid "https://foursquare.com/venue/215159"))
	{:total_row_count 1, :included_rows 1, :version 3, :status "ok"}

	> (meta (fun/get-urls "97598010-433f-4946-8fd5-4a6dd1639d77" :limit 12))
	{:total_row_count 66, :included_rows 12, :version 3, :status "ok"}

# Handling Bad Responses

Funnyplaces uses the Slingshot library to indicate API errors. If Funnyplaces encounters an API error, a Slingshot stone called funnyplaces-error will be thrown.

The funnyplaces-error will contain information about the error, including the server response code and any options you used to create the query.

Example:

	;  (:import [funnyplaces.api funnyplaces-error])
	
	(try+
	  (fun/fetch :places :filters {:factual_id "97598010-433f-4946-8fd5-4a6dd1639d77" :BAD :PARAM!})
	  (catch funnyplaces-error {code :code message :message opts :opts}
	    (println "Got bad resp code:" code)
	    (println "Message:" message)
	    (println "Opts:" opts)))

# An Example of Tying Things Together

Let's create a simple function that finds Places close to a lat/lng, with "cafe" in their name:

	(defn nearby-cafes
	  "Returns up to 12 cafes within 5000 meters of the specified location."
	  [lat lon]
	  (fun/fetch :places
	         :q "cafe"
	         :filters {:category {:$eq "Food & Beverage"}}
	         :geo {:$circle {:$center [lat lon]
	                         :$meters 5000}}
	         :include_count true
	         :limit 12))

Using our function to get some cafes:

	> (def cafes (nearby-cafes 34.06018 -118.41835))

Let's peek at the metadata:

	> (meta cafes)
	{:total_row_count 26, :included_rows 12, :version 3, :status "ok"}

Ok, we got back a full 12 results, and there's actually a total of 26 cafes near us. Let's take a look at a few of the cafes we got back:

	> (map :name (take 3 cafes))
	("Aroma Cafe" "Cafe Connection" "Panini Cafe")

That first one, "Aroma Cafe", sounds interesting. Let's see the details:

	> (clojure.contrib.pprint/pprint (first cafes))
	{:status "1",
	 :country "US",
	 :longitude -118.423421,
	 :factual_id "eb67e10b-b103-41be-8bb5-e077855b7ae7",
	 :name "Aroma Cafe",
	 :postcode "90064",
	 :locality "Los Angeles",
	 :latitude 34.039792,
	 :region "CA",
	 :address "2530 Overland Ave",
	 :website "http://aromacafe-la.com/",
	 :tel "(310) 836-2919",
	 :category "Food & Beverage"}

So I wonder what Yelp has to say about this place. Let's use Crosswalk to find out. Note that we use Aroma Cafe's :factual_id from the above results...

	> (fun/crosswalk :factual_id "eb67e10b-b103-41be-8bb5-e077855b7ae7" :only "yelp")
	({:factual_id "eb67e10b-b103-41be-8bb5-e077855b7ae7",
	  :namespace :yelp,
	  :namespace_id "AmtMwS2wCbr3l-_S0d9AoQ",
	  :url "http://www.yelp.com/biz/aroma-cafe-los-angeles"})
	  
That gives me the yelp URL for the Aroma Cafe, so I can read up on it on Yelp.com.

Of course, Factual supports other Crosswalked sources besides Yelp. If you look at each row returned by the <tt>crosswalk</tt> function, you'll see there's a <tt>:namespace</tt> in each one. Let's find out what namespaces are available for the Aroma Cafe:

	> (map :namespace (fun/crosswalk :factual_id "eb67e10b-b103-41be-8bb5-e077855b7ae7"))
	(:merchantcircle :urbanspoon :yahoolocal :foursquare :yelp ... )

Let's create a function that takes a :factual_id and returns a hashmap of each valid namespace associated with its Crosswalk URL:

	(defn namespaces->urls [factid]
	  (into {} (map #(do {(:namespace %) (:url %)})
	    (fun/crosswalk :factual_id factid))))

Now we can do this:

	> (namespaces->urls "eb67e10b-b103-41be-8bb5-e077855b7ae7")
	{:merchantcircle   "http://www.merchantcircle.com/business/Aroma.Cafe.310-836-2919",
 	 :urbanspoon	   "http://www.urbanspoon.com/r/5/60984/restaurant/West-Los-Angeles/Bali-Place-LA",
 	 :yahoolocal       "http://local.yahoo.com/info-20400708-aroma-cafe-los-angeles",
 	 :foursquare       "https://foursquare.com/venue/46f53d65f964a520f04a1fe3",
 	 :yelp             "http://www.yelp.com/biz/aroma-cafe-los-angeles",	
	 ... }
