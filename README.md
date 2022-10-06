Like I said in the [StackOverflow answer](https://stackoverflow.com/a/73935713/3645650), as an alternative to the Geolocation browser API, we should be able to reverse IP geocoding the user location. A free, reliable option seems to be the [geoplugin](https://www.geoplugin.com/) API.

We will be retriving all user's currently active sessions and get the user's associated IP. We can then use each ip to retrieve each associated location, and push the whole thing to a custom user meta.

We hook the whole thing to the `wp_login` action hook which fires after the user has successfully logged in.

```php
<?php

add_function( 'wp_login', function ( $user_login, $user ) {

    /**
     * In our case wp_get_all_sessions() won't work as it rely on get_current_user_id(), and at that point in time get_current_user_id() is not yet "accessible".
     * The alternative is to mirror the wp_get_all_sessions() function and use the second parameter from the wp_login hook which let us retrieve the WP_User object of the logged-in user.
     * We can then access the user ID through $user-ID.
     * 
     * @see https://developer.wordpress.org/reference/functions/wp_get_all_sessions/.
     */
    $manager = WP_Session_Tokens::get_instance( $user->ID );
    
    $sessions = $manager->get_all();

    //Plucks the ip field out of each sessions's array in an array.
    $ips = wp_list_pluck( $sessions, 'ip' );

    $locations = array();

    foreach ( $ips as $ip ) {

        if ( ! empty( $ip ) ) {

            /**
             * Reverse IP geocoding location.
             * 
             * @see https://www.geoplugin.com/webservices/php.
             * @see https://www.php.net/manual/en/function.file-get-contents.php.
             */
            $api = 'http://www.geoplugin.net/php.gp?ip=' . $ip;

            if ( @fopen( $api, 'r' ) ) {

                //Returns the file in a string.
                $location = file_get_contents( $api );

                //Creates a PHP value from a stored representation.
                $location = unserialize( $location );
    
                //Push each ip's location to a new array.
                array_push( $locations, $location );
    
                return $locations;

                update_user_meta( $user->ID, '_user_sessions_locations', $locations );

            };

        };

    };

}, 10, 2 );
```

On the front end we can retrieve the user meta through `get_user_meta()`. The last key in the array is the last session.

```
<?php

$user_id = get_current_user_id();

$meta_value = get_user_meta( $user_id, '_user_sessions_locations' );

var_dump( $meta_value );

var_dump( end( $meta_value ) );
```
