Converting existing Fields to MeasurementField
==============================================

Field is already in the 'Standard' Unit
---------------------------------------

This is the trivial case. We can just replace the existing field with a
measurement field::

    from django_measurement.models import MeasurementField
    from measurement.measures import Volume
    from django.db import models

    class BeerConsumptionLogEntry(models.Model):
        name = models.CharField(max_length=255)
        volume = models.FloatField(
            help_text="Volume of beer consumed in litres"
        )  # Field to be converted

        def __str__(self):
            return '%s of %s' % (self.name, self.volume)

goes to::

    from django_measurement.models import MeasurementField
    from measurement.measures import Volume
    from django.db import models

    class BeerConsumptionLogEntry(models.Model):
        name = models.CharField(max_length=255)
        volume = MeasurementField(measurement=Volume)

        def __str__(self):
            return '%s of %s' % (self.name, self.volume)

Because ``Volume`` uses the litre as its ``STANDARD_UNIT`` everything is happy.
Run ``makemigrations`` and everyone is happy.

Field in non 'Standard' Units
-----------------------------

In this case it's necessary to first add a data migration that converts the
field into standard units. Say we have a model like this::

    from django_measurement.models import MeasurementField
    from measurement.measures import Volume
    from django.db import models

    class BeerConsumptionLogEntry(models.Model):
        name = models.CharField(max_length=255)
        volume = models.FloatField(
            help_text="Volume of beer consumed in hectolitres"
        )  # Field to be converted

        def __str__(self):
            return '%s of %s' % (self.name, self.volume)

We first need to convert the volume field to liters like so::

    litres_per_hectolitre = 100

    def convert_field(apps, _):
        BeerConsumptionLogEntry = apps.get_model("beer", "BeerConsumptionLogEntry")

        BeerConsumptionLogEntry.all_objects.update(
            volume=Subquery(
                BeerConsumptionLogEntry.objects.filter(id=OuterRef("id"))
                .annotate(converted_volume=F("volume") * litres_per_hectolitre)
                .values("converted_volume")[:1]
            )
        )


    def convert_field_reverse(apps, _):
        BeerConsumptionLogEntry = apps.get_model("beer", "BeerConsumptionLogEntry")

        BeerConsumptionLogEntry.all_objects.update(
            volume=Subquery(
                BeerConsumptionLogEntry.objects.filter(id=OuterRef("id"))
                .annotate(converted_volume=F("volume") / litres_per_hectolitre)
                .values("converted_volume")[:1]
            )
        )

And then hook those two functions up into a RunPython migration as per usual.
Having an actual reverse migration here is critical. It is also be critical
that it is legitimately the inverse. Multiplying and or dividing by the exact
same constant ensures this.

Now you can just convert the field to a measurement field as above.
