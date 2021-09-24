Django Graphene Query Ordering
==============================

Summary
-----------------

The django-graphene docus on filtering & ordering don't seem completely up to date. 
(https://docs.graphene-python.org/projects/django/en/latest/filtering/)
A quick reference below on how it does work.


React
-----------------------------

The TinyMCE react module is used (npm install @tinymce/tinymce-react)

In the frontend an Editor component can be instructed to load it like so:

.. code-block:: python

    from django_filters import FilterSet, OrderingFilter


    # For a single field
    class ScheduleEventTicketScheduleItemFilter(FilterSet):
        class Meta:
            model = ScheduleEventTicketScheduleItem
            fields = ['schedule_event_ticket', 'schedule_item', 'included']

        order_by = OrderingFilter(
            fields=(
                ('schedule_item__date_start', 'date_start'), # order by field, display name (used in query)
            )
        )
    # Single field end

    # For multiple fields
    class CustomOrderingFilter(OrderingFilter):
    """
    A Custom ordering filter to allow ordering by multiple fields
    """
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.extra['choices'] += [
            ('date_start', 'Date start'),
            ('-date_start', 'Date start (descending)'),
        ]

    def filter(self, qs, value):
        # OrderingFilter is CSV-based, so `value` is a list
        if any(v in ['date_start', '-date)start'] for v in value):
            # sort queryset by relevance
            return qs.order_by('schedule_item__date_start', 'schedule_item__time_start')

        return super().filter(qs, value)


    class ScheduleEventTicketScheduleItemFilter(FilterSet):
        class Meta:
            model = ScheduleEventTicketScheduleItem
            fields = ['schedule_event_ticket', 'schedule_item', 'included']

        order_by = CustomOrderingFilter()

    # Multiple fields end


    # The setting of filterset_class is the same in all cases
    class ScheduleEventTicketScheduleItemNode(DjangoObjectType):
        class Meta:
            model = ScheduleEventTicketScheduleItem
            filterset_class = ScheduleEventTicketScheduleItemFilter
            interfaces = (graphene.relay.Node,)

        @classmethod
        def get_node(self, info, id):
            user = info.context.user
            require_login_and_permission(user, 'costasiella.view_scheduleeventscheduleitem')

            return self._meta.model.objects.get(id=id)

    
Query example of how we can now use orderBy in a subQuery

.. code-block:: javascript

    import { gql } from "@apollo/client"


    export const GET_SCHEDULE_EVENT_TICKET_QUERY = gql`
        query ScheduleEventTicket($id: ID!) {
            scheduleEventTicket(id:$id) {
            id
            name
            price
            priceDisplay
            totalPrice
            totalPriceDisplay
            description
            isSoldOut
            isEarlybirdPrice
            ticketScheduleItems(included: true, orderBy: "scheduleItem_DateStart") {
                pageInfo {
                    hasNextPage
                    hasPreviousPage
                    startCursor
                    endCursor
                }
                edges {
                    node {
                        id
                        included
                        scheduleItem {
                        name
                        dateStart
                        timeStart
                        timeEnd
                        organizationLocationRoom {
                            organizationLocation {
                                name
                                }
                            }
                        }
                    }
                }
            }
                scheduleEvent {
                    id
                    name
                }
            }
        }
    `
