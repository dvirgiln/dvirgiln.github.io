---
title: Nested form validation using Scala Cats
description: Usage of Applicative and Validated Cats type classes to perform a multi
  level form validation.
layout: post
featured: images/cats.png
---

One of the most common usages of the Applicative Cats type class is the form validation because of its parallel computation nature. While in a monad everything is computed sequencially, in an applicative functor the computation is not restricted to be executed in any particular order. It is executed in parallel. While in a monad if during the execution of the monad it fails, the execution is stopped, in an applicative due to its nature, the computation continues. 

With Applicatives we are able to return all the errors that one form has, while if we would use a monad to try the same, we could just return the first error. That's why Applicative functors are so useful for a form validation. The validation of every field can be done in parallel and at the end the result of the validation can be joined and merged at the end of the process.

You can check in google about form validation examples using Cats Applicative and there are a few of good posts, but it is quite difficult to find a good example about a nested validation form. So, the majority of the examples you can find just a one level form, like for instance an Address:

```
case class Address(street: String, number: String, postalCode: String, county: Option[String], country: String)
```

But in the real world forms are quite complex, and normally the structures contain three or four nested objects. For instance:

```
  case class Property(listing: Listing) extends UUIDable { def id: Option[String] = listing.id }
  case class Listing(id: Option[String], contact: Contact, address: Address, location: GeoLocation)
  case class Contact(phone: String, formattedPhone: String)
  case class Address(address: String, postalCode: String, countryCode: String, city: String, state: Option[String], country: String)
  case class GeoLocation(lat: String, lng: String)
```
In the previous code it has been modeled a hotel property listing.  Obviously we could have model a form validator based on one big validator that validates all the different values of the nested elements. But the solution is not quite clean and its quite coupled. For instance the address case class could be used in other parts of our code as part of other class.

The main objective of this article is to show a good example that solves the problem of a composition of validators using Cats Applicative functor combined with the Validated functor.

Let's review what it is exactly the syntax of the Applicative functor:

```
trait Applicative[F[_]] extends Apply[F]{
  def pure[A](x: A): F[A]
	  override def map[A, B](fa: F[A])(f: A => B): F[B] = ap(pure(f))(fa)
}
```

It contains 2 functions:
* *Pure* containerize an object of type A into F[A}. If our applicative is applied over the monad List, it will add the element to the List. The same apply to Option or Future. 
* *Map*: as applicative is a functor it should have the map operation. We can see that to implement the map operation it has been done in terms of pure and *ap*. The operation *ap* is the most important operation of Applicative. It is implement in the class Apply that this class is mixing.

This is the syntax of the Apply trait:

```
trait Apply[F[_]] extends Functor[F]  {
  def ap[A, B](ff: F[A => B])(fa: F[A]): F[B]
}
```
As we can see it has two curried parameters. The first contains a function inside of our container F[A =>B] and the second is the containerized first element of the function. Weird, isn't it? Well you will see the usage more clearly once we start with our example.

## Let's do it

We want to create a composite validator using Cats for the following form:

```
  case class Property(listing: Listing) extends UUIDable { def id: Option[String] = listing.id }
  case class Listing(id: Option[String], contact: Contact, address: Address, location: GeoLocation)
  case class Contact(phone: String, formattedPhone: String)
  case class Address(address: String, postalCode: String, countryCode: String, city: String, state: Option[String], country: String)
  case class GeoLocation(lat: String, lng: String)
```
We can start with the implementation of the *AddressValidator*:

```
       ...
      /*
       * No real validation has been done in the address
       */
      def validateAddress(address: String): F[String] = address.pure[F]

      /*
       * No real validation has been done in the postal code.
       */
      def validatePostalCode(postalCode: String): F[String] = postalCode.pure[F]

      /*
       * I assumed here that the country code is a 2 characters code.
       */
      def validateCountryCode(countryCode: String): F[String] =
        if (countryCode.size == 2) countryCode.pure[F]
        else A.raiseError(mkError(AddressNonCorrectCountryCodeError))

      /*
       * Validates that the state property is assigned correctly. I assumed that US is the only country with the state property.
       * In case the country code is US then it should have the state property. In case it is any other country the state should be None.
       */
      def validateState(address: Address): F[Option[String]] = {
        if (address.countryCode == "US") address.state match {
          case None => A.raiseError(mkError(AddressShouldContainStateError))
          case Some(_) => address.state.pure[F]
        }
        else {
          address.state match {
            case None => address.state.pure[F]
            case Some(_) => A.raiseError(mkError(AddressShouldNotContainStateError))
          }
        }
      }

      /*
       * It has been validated that the city is not empty. Mandatory property.
       */
      def validateCity(city: String): F[String] =
        if (city.isEmpty)
          A.raiseError(mkError(AddressCityIsEmptyError))
        else
          city.pure[F]

      /*
       * It has been validated that the country is not empty. Mandatory property.
       */
      def validateCountry(country: String): F[String] =
        if (country.isEmpty)
          A.raiseError(mkError(AddressCountryIsEmptyError))
        else
          country.pure[F]

      /*
       * Main entry point, it provides the validation of the Address class
       * sequencing the validation of the object using Cats applicative
       */
      def validate(address: Address): F[Address] = {
        (Address.apply _).curried.pure[F].
          ap(validateAddress(address.address)).
          ap(validatePostalCode(address.postalCode)).
          ap(validateCountryCode(address.countryCode)).
          ap(validateCity(address.city)).
          ap(validateState(address)).
          ap(validateCountry(address.country))
      }
    }

   ...
}
```

Considerations of the previous code:
* Line 58: it receives an address as a parameter and it returns a monad of address, in this case we do not know the kind of monad until this class is instantiated. 
* Line 59: we are building an address as a curried nested function of parameters. What the hell *(Address.apply _).curried* does? It converts a constructor case class Address into a nested function:
```   
  case class Address(address: String, postalCode: String, countryCode: String, city: String, state: Option[String], country: String) 
```
to 
```
	String => String => String =>String => Option[String] => String => Address
```
Magic? No, Scala.
* Continuation of line 59:  so, we have a nested function and we applied the pure function from *Applicative*, so we have a:
```
F[String => String => String =>String => Option[String] => String => Address]
```
* Looks like the first parameter of the *ap* function from Aplicative class, isn't it?
* From line 60 to line 65: we are able to apply the method ap to the function F[String => String ...] and construct our F[Address].
* You can realize that when we construct our F[Address] we are calling a method validateCountry, validatePostalCode... These functions check the validity of each attribute and return a F[]. 
* In all the validation methods, you can see that to construct a valid value we just need to call the F[].pure to containerize the value.
* In case that there is an error, we can raise it *A.raiseError(mkError*

In the Address Validator we have seen the root of our validators, but what about the validator that is a composition of other validators. We need to make use of another class provided by Cats called **Validated**. Let's understand Validated first:

```
sealed abstract class Validated[+E, +A] extends Product with Serializable {

  def fold[B](fe: E => B, fa: A => B): B =
    this match {
      case Invalid(e) => fe(e)
      case Valid(a) => fa(a)
    }

  def isValid: Boolean = fold(_ => false, _ => true)
  def isInvalid: Boolean = fold(_ => true, _ => false)
  ...
```


Validated is not a monad. If you try to execute the flatMap operation, you can see that doesn't exist. Validated is an Applicative functor that allows parallel computation and acummulation of errors. This is exactly what we need in our form validation. In other cases we may want to have a monadic validation, and fail with the first error. In that case we could use the *Either* monad.

Validated has 2 type parameters E and A. The E type correspond to the kind of error that is returned in case of error. The A type parameter correspond to the return type in case of a valid response. Similar to the Either type parameters.

To help us, Cats provides an extension type of the main Validated type, that we are going to use in our example:

```
  type ValidatedNel[+E, +A] = Validated[NonEmptyList[E], A]
```
The ValidatedNel models the error accumulation use case, that it is exactly what we need.

We can see part of the implementation of the Listing Validator. 

```
    ...
      def validateId(idOpt: Option[String], emptyID: Boolean): Validated[PropertyValidationError, Option[String]] = {
        emptyID match {
          case true => idOpt match {
            case None => Valid(idOpt)
            case Some(_) => Invalid(IdNonEmptyError)
          }
          case false => idOpt match {
            case None => Invalid(IdEmptyError)
            case Some(id) => Try(UUID.fromString(id)).isSuccess match {
              case true => Valid(idOpt)
              case false => Invalid(NonUUIDFormatError)
            }
          }
        }

      }

      def validateGeoLocation(location: GeoLocation): Validated[PropertyValidationError, GeoLocation] = {
        val tryLat = Try(location.lat.toDouble)
        val tryLong = Try(location.lng.toDouble)
        if (tryLat.isSuccess && tryLong.isSuccess)
          Valid(location)
        else Invalid(GeoLocationNonExistingError)
      }

      /*
        Validates that the property has a valid format. It calls internally another validators and join the result.
        @param property -> The property
        @param emptyID -> Indicates to the validator if the id should be empty or not. In case the id shouldnt be empty, checks that it has UUID format.
       */
      def validate(listing: Listing, emptyID: Boolean): ValidatedNel[PropertyValidationError, Listing] = {
        Apply[ValidatedNel[PropertyValidationError, ?]].map4(
          validateId(listing.id, emptyID).toValidatedNel,
          ContactValidator.validate(listing.contact),
          AddressValidator.validate(listing.address),
          validateGeoLocation(listing.location).toValidatedNel
        ) {
            case (id, contact, address, location) => Listing(id, contact, address, location)
          }
      }

      ...
```

Considerations about the previous code:
* This validator contains inline validations, like the geolocation validation and the id validation.
* It is composed as well of external validators, like the ContactValidator or the AddressValidator.
* Line 32: this is the main function of the validation. It creates an applicative using the Apply constructor. The container in this case is ValidatedNel, to aggregate *PropertyValidationError*s. 
* Line 33 : we can see that it is applied the function map4, that belongs to SemiGroupal. SemiGroupal captures the idea of composing independent effectful values. This is exactly what we need in our application. 
* Line 19: the validateGeoLocation returns a Valid or Invalid value. We can see that the function returns a Validated type. 

## Source Code
The complete source code can be found in this link:

[https://github.com/dvirgiln/listing-api](https://github.com/dvirgiln/listing-api)

The source code contains a Rest API that applies this form validation. 

Apart from it, you can see the junit tests that validate the different parts of the application.

## Conclusion

Cats, Monads, Applicative, Semigroupal... all of this could seem a bit cryptic and chaotic at first. But well applied it can help you and solve complex scenarios.

I hope this post about Cats and Scala could help you integrating Cats in your codebase.