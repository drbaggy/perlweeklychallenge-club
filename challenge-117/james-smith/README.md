# Perl Weekly Challenge #117

You can find more information about this weeks, and previous weeks challenges at:

  https://perlweeklychallenge.org/

If you are not already doing the challenge - it is a good place to practise your
**perl** or **raku**. If it is not **perl** or **raku** you develop in - you can
submit solutions in whichever language you feel comfortable with.

You can find the solutions here on github at:

https://github.com/drbaggy/perlweeklychallenge-club/tree/master/challenge-117/james-smith/perl

# Task 1 - Missing Row

***You are given text file with rows numbered 1-15 in random order but there is a catch one row in missing in the file. Write a script to find the missing row number.***

## The solution

It would first seem we would need to collect a complete list of line numbers - but that is not the case.

If we have a file with `N` rows, we now that the sum of the line numbers is `N*(N+1)/2`.

So to find the one that is missing we just sum the line numbers and take it from `N*(N+1)/2`.

If `T` is the total of the line numbers and `n` is the number of lines read then:

`N = n+1` so `T` + `missing number` = `(n+1)(n+2)2`

```perl
sub splitnum {
  my( $N, $T ) = ( 1, 0 );
  open my $fh, q(<), shift;
  ++$N && ( $T += substr $_, 0, index $_, q(,) ) while <$fh>;
  close $fh;
  return $N * ( $N + 1 ) / 2 - $T;
}
```

# Task 2 - Find Possible Paths

***You are given size of a triangle. Write a script to find all possible paths from top to the bottom right corner. In each step, we can either move horizontally to the right (H), or move downwards to the left (L) or right (R).***

## The solution

The output of this script will be large - especially for larger sizes. We will look at the "count" only version lately. But e.g for size 10 - there are 1,037,718 routes and size 20 - there are 17,518,619,320,890 routes.

For dumping the routes - this lends itself to a recursive solution:

```perl
sub triangle {
  my( $size, $offset, $route ) = @_;
  ( say $route.( 'H' x $offset ) ) && return     unless $size;
  triangle( $size - 1, $offset + 1, $route.'L' );
  triangle( $size - 1, $offset,     $route.'R' );
  triangle( $size,     $offset - 1, $route.'H' ) if $offset;
}
```

**Notes:**

`$offset` is the distance from the right hand side of the triangle - so moving left (`L`)
increments `$offset` and moving horizontally (`H`) decrements `$offset`.

If we get to the bottom row - we short-cut the recursion by just including an `H` for
every point we are to the left of the corner (which just happens to be `$offset`)...

We don't "collect" routes in a data structure and then print them all at the
end, but instead render directly from within the subroutine. For `$N` larger than
`10` the memory requirements for storing this information increases significantly,
so this code is limited purely by disk rather than memory.

### Now the counts... Schröder numbers

*It's amazing what you find out about when you google the answers you get!*

Due to the memory/storage issues we can change the problem to one of counting rather than listing...
The first approach is to just convert the `triangle` method above to count - we introduce a cache
as well to improve performance.

```perl
sub schröder_cache_array {
   my($size,$offset) = @_; $offset ||=0;
   return $size
        ? ( $cache[$size][$offset] ||=
              schroder_cache_array( $size - 1, $offset + 1 ) #L
            + schroder_cache_array( $size - 1, $offset     ) #R
            + ( $offset ? schroder_cache_array( $size, $offset - 1 ) : 0 )
          )
        : 1;
}
```

But as we've said before recursion is a curse - but we notice that
```
       T0,m = 1                          
  Sn = Tn,0 = Tn-1,0 + Tn-1,1
       Tn,m = Tn-1,m + Tn-1,m+1 + Tn,m-1
```

We can use that to define each row of a triangle with `Sn` as the last
value.

```perl
sub schröder_non_recursive {
  my $size = shift;
  my @x = map {1} 0..$size;
  foreach my $s (1..$size) {
    my @y = $x[1] + $x[0];
    push @y, $x[$_+1] + $x[$_] + $y[-1] foreach 1 .. $size-$s;
    @x=@y;
  }
  return $x[0];
}
```

We again use the row "flip" method as we only need one row and the previous
one to get values... Avoids having to allocate more memory for the whole
triangle.

### The quickest counter!

Googling for `2, 6, 22, 90, 394` came up with https://en.wikipedia.org/wiki/Schröder_number, this has
lots of information about uses of this sequence. As well as giving the non-recursive relation above it
also gives a faster (about twice as fast as above) solution - as Scrhöder numbers can be written as a
recurrence relation. Writing this in perl gives us, where @S = is the array of Scrhöder numbers.

```perl
sub schröder_recurrence_rel {
  my( $size, @S ) = ( shift, 1, 2 );
  foreach my $n (2..$size) {
    $S[ $n ]  =        3 * $S[ $n - 1      ];
    $S[ $n ] += $S[ $_ ] * $S[ $n - 1 - $_ ] foreach 1..$n-2;
  }
  return $S[ $size ];
}
```
