---
layout: post
title: "Plugging in implementation details"
date: 2025-02-16 22:17:00 -0000
categories: CATEGORY-1 CATEGORY-2
---
# Plugging in implementation details

**Software is easier to change when we can plugin implementation into our business rules.**

[comment]: # TODO: change this image to match business rules/logic
![Image of implementation details being plugged into business rules]({{ site.baseurl }}/assets/images/2025-02-16-plugging-in-implementation-details/business-logic-implementation.jpg)

## A narrative
Imagine you’ve joined a company that has a web app for ordering books. They are currently using Google Analytics to track events in the website and the analytics team wants to move to Snowplow.

Here’s how you might send an `order_submitted` event to Google Analytics:
```javascript
window.gtag('event', 'order_submitted', order);
```

Here’s how you might send the same event to Snowplow:
```javascript
window.snowplow('trackSelfDescribingEvent', {
    event: 'iglu:com.acme_company/order_submitted/jsonschema/1-0-0',
    data: order,
});
```

You start looking through the web app code to see what will need to change.

### Separating concerns

Somewhere on the site there’s a button with the words *Submit Order*. When a user clicks this button, the web app sends the order to a backend API and tracks this as an event in Google Analytics. Here’s what that code might look like:

```javascript
import { useCallback } from "react";
import { sendOrderToBackend } from '@/services/backend';
import { Order } from "@/models/order";


export const SubmitOrderButton: React.FC<{ order: Order }> = ({ order }) => {
    const submitOrder = useCallback(async () => {
        await sendOrderToBackend(order);
        window.gtag('event', 'order_submitted', order);
    }, [order]);


    return (
        <button
            onClick={submitOrder}
        >
            Submit Order
        </button>
    );
};
```

The test for this might look something like this:
```javascript
describe('SubmitOrderButton', () => {
    let sendOrderToBackend: MockInstance;
    let gtag: MockInstance;


    beforeEach(() => {
        gtag = vi.fn();
        vi.stubGlobal('gtag', gtag);
        sendOrderToBackend = vi.spyOn(backend, 'sendOrderToBackend');
    });


    afterEach(() => {
        sendOrderToBackend.mockRestore();
    });


    describe('clicking the button', () => {       
        it('sends a GA order_submitted event', async () => {
            render(<SubmitOrderButton order={testOrder} />);
            const button = screen.getByRole('button');


            fireEvent.click(button);


            await waitFor(() => {
                expect(gtag)
                    .toHaveBeenCalledExactlyOnceWith(
                        'event',
                        'order_submitted',
                        testOrder,
                    );
            });
        });
    });
});
```

To change this to use Snowplow, you’d have to change the component and the test to call Snowplow correctly. Of course, there are 100 different components in the web app that use `window.gtag`, so you have 100 components and 100 tests to change. Great.

Your life would be a lot easier if the `window.gtag` details were hidden inside a `sendGaEvent` function, e.g.
```javascript
import { useCallback } from "react";
import { sendOrderToBackend } from '@/services/backend';
import { Order } from "@/models/order";
import { sendGaEvent } from "@/services/ga";


export const SubmitOrderButton: React.FC<{ order: Order }> = ({ order }) => {
    const submitOrder = useCallback(async () => {
        await sendOrderToBackend(order);
        sendGaEvent('order_submitted', order);
    }, [order]);


    return (
        <button
            onClick={submitOrder}
        >
            Submit Order
        </button>
    );
};
```
```javascript
describe('SubmitOrderButton', () => {
    let sendGaEvent: MockInstance;
    let sendOrderToBackend: MockInstance;


    beforeEach(() => {
        sendGaEvent = vi.spyOn(ga, 'sendGaEvent');
        sendOrderToBackend = vi.spyOn(backend, 'sendOrderToBackend');
    });


    afterEach(() => {
        sendGaEvent.mockRestore();
        sendOrderToBackend.mockRestore();
    });


    describe('clicking the button', () => {       
        it('sends a GA order_submitted event', async () => {
            render(<SubmitOrderButton order={testOrder} />);
            const button = screen.getByRole('button');


            fireEvent.click(button);


            await waitFor(() => {
                expect(sendGaEvent)
                    .toHaveBeenCalledExactlyOnceWith('order_submitted', testOrder);
            });
        });
    });
});
```

If the code was written like that then you could globally find-and-replace `sendGaEvent` with `sendSnowplowEvent` (remembering to replace the import paths too). You could hide the details of sending an event to Snowplow inside your new `sendSnowplowEvent` function - there’d still be 200 files changed, but the only meaningful change to review is the new `sendSnowplowEvent` function.

It would be even easier if the function was called `sendEvent` in the first place. Then you wouldn’t have to rename anything, you could just replace the internals of the `sendEvent` function.