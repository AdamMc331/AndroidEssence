+++
date = '2026-04-25'
draft = false
title = 'Managing Multi Table Inserts With Room'
+++

A properly relational database may have a type that appears referenced multiple times. In my [SpaceNerd](https://github.com/AdamMc331/SpaceNerd) playground app, that is the Country object. An Agency has a country, and an Agency is a dependency of a Launch, of a Space Station, and likely more types to come. 

How do I enforce that any time I insert an Agency, I also insert the Countries associated with it? That's what this blog post sets out to explore, and the answer is much simpler than I thought when I started.

<!--more-->

---

## Previous Setup

The way I was used to working with Room, is to have one [Dao](https://developer.android.com/training/data-storage/room/accessing-data) interface for each table, or at least domain object type that I want to persist. That lead me to have a setup like this:

```kotlin
@Dao
interface SpaceStationDao {
    @Insert
    suspend fun insertCountry(dto: RoomCountryDTO)

    @Insert
    suspend fun insertAgency(dto: RoomAgencyDTO)

    @Insert
    suspend fun insertAgencyCountryCrossReference(dto: RoomAgencyCountryCrossRefDTO)

    @Insert
    suspend fun insertSpaceStation(dto: RoomSpaceStationDTO)

    @Transaction
    suspend fun insertDomainSpaceStation(station: SpaceStation) {
        insertSpaceStation(station)

        for (agency in station.agencies) {
            insertAgency(agency)

            for (country in agency.countries) {
                insertCountry(country)

                insertAgencyCountryCrossReference(
                    agencyId = agency.id,
                    countryId = country.id,
                )
            }
        }
    }
}

@Dao
interface LaunchDao {
    @Insert
    suspend fun insertCountry(dto: RoomCountryDTO)

    @Insert
    suspend fun insertAgency(dto: RoomAgencyDTO)

    @Insert
    suspend fun insertAgencyCountryCrossReference(dto: RoomAgencyCountryCrossRefDTO)

    @Insert
    suspend fun insertLaunch(dto: RoomLaunchDTO)

    @Transaction
    suspend fun insertDomainLAunch(launch: Launch) {
        insertLaunch(launch)

        val agency = launch.agency

        insertAgency(agency)

        for (country in agency.countries) {
            insertCountry(country)

            insertAgencyCountryCrossReference(
                agencyId = agency.id,
                countryId = country.id,
            )
        }
    }
}
```

The above is psuedocode for simplicity, but you can see that the country logic is duplicated in each Dao. 

## The Problem

I would love for these to reference each other, but dependency injection is not an option with interfaces, I have no way to call one Dao from another.

One solution I explored was to create a wrapper class, that would be responsible for inserting objects that have dependencies:

```kotlin
class AgencyPersister(
    val agencyDao: AgencyDao,
    val countryDao: CountryDao,
) {
    suspend fun insertAgency(agency: Agency) {
        // Same insert logic as above for agency and its countries
    }
}
```

Then, any time I needed this, I would reference in a different persister type class:

```kotlin
class SpaceStationPersister(
    val agencyPersister: AgencyPersister,
    val spaceStationDao: SpaceStationDao,
) {
    suspend fun insertStation(station: SpaceStation) {
        // Call agency persister to insert agency, and implicitly countries
        // Then call space station Dao to insert station
    }
}
```

I didn't like this solution, though, as it lead to a mix of Dao interface and persister classes that may be hard to keep track of. So, I kept digging, and finally remembered a super basic principle. 

## Inheritance

Interfaces cannot call each out, but they _can_ inherit from each other. This means, any base logic like inserting an agency and its countries could go into their own interface:

```kotlin
interface BaseAgencyDao {
    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertOrIgnoreCountry(
        country: RoomCountryDTO,
    )

    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertOrIgnoreAgencyCountryCrossRef(
        crossRef: RoomAgencyCountryCrossRefDTO,
    )

    @Upsert
    suspend fun upsertAgency(
        agency: RoomAgencyDTO,
    )

    private suspend fun insertAgencyCountry(
        country: Country,
        agencyId: String,
    ) {
        val countryDto = RoomCountryDTO(country)
        insertOrIgnoreCountry(countryDto)

        val crossRef = RoomAgencyCountryCrossRefDTO(
            agencyId = agencyId,
            countryId = country.id,
        )

        insertOrIgnoreAgencyCountryCrossRef(crossRef)
    }

    @Transaction
    suspend fun upsertDomainAgency(
        agency: Agency,
    ) {
        for (country in agency.countries) {
            insertAgencyCountry(
                country = country,
                agencyId = agency.id,
            )
        }

        val agencyDto = RoomAgencyDTO(agency)
        upsertAgency(agencyDto)
    }
}
```

Now, any types that have an agency property, like a SpaceStation, can inherit from this base interface:

```kotlin
@Dao
interface SpaceStationDao : BaseAgencyDao {
    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertSpaceStationAgencyCrossRef(
        stationAgencyCrossRefDTO: RoomSpaceStationAgencyCrossRefDTO,
    )

    @Upsert
    suspend fun upsertSpaceStation(
        station: RoomSpaceStationDTO,
    )

    @Transaction
    suspend fun upsertDomainSpaceStation(
        station: SpaceStation,
    ) {
        for (agency in station.agencies) {
            upsertDomainAgency(agency)

            val crossRef = RoomSpaceStationAgencyCrossRefDTO(
            spaceStationId = spaceStationId,
            agencyId = agency.id,
        )

        insertSpaceStationAgencyCrossRef(crossRef)
        }

        val stationDto = RoomSpaceStationDTO(station)
        upsertSpaceStation(stationDto)
    }
}
```

Note that the SpaceStationDao is now much smaller, and the logic inside this Dao only pertains to what is necessary to persist a SpaceStation itself, and it calls the other interface via inheritance to persist an Agency, and any sub dependencies.

This can go even further; We could move the above to a `BaseSpaceStationDao`, so in the future, `ExpeditionDao` (which is a type with an associated space station) can just inherit from that, and all the internals on how a space station is persisted are fully abstracted away.

A lot of research went in to something that boiled down to a basic OOP principle, but if helped me, hopefully it helped you too.  

For more database and other general Android content, make sure you're following me over on [Twitch](https://twitch.tv/adammc) and [YouTube](https://youtube.com/adammcneilly)!