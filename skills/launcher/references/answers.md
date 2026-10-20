# Launch answer keys

`launch_start` and `launch_answer` take `answers` as `{key: value}`. Only these keys exist; Adset refuses any other. A value can be the option's value or its name as shown (a name contained in exactly one option also works). Use them only for what the user actually said; Adset asks for the rest.

## Tasks

| Key | Value |
|---|---|
| `platforms` | `["meta"]`, `["google"]`, `["applovin"]` or several |
| `relaunch` | `include` (launch again tasks already launched) or `exclude` |
| `exclude_tasks` | task ids to leave out |

## Meta (`meta_…`)

| Key | Value |
|---|---|
| `meta_template` | Launch Template name or id, or `none` |
| `meta_ad_account` | ad account (`act_…` or its name) |
| `meta_creative` | `field_mapping` (texts from the task) or a Creative Template name / id |
| `meta_destination` | `existing_ad_set`, `new_ad_set`, `new_campaign` |
| `meta_campaign` | campaign name or id |
| `meta_ad_set` | existing Ad Set name or id |
| `meta_campaign_name` | name of a new campaign |
| `meta_ad_set_grouping` | `shared` (one new Ad Set for all tasks) or `separate` (one per task) |
| `meta_ad_set_naming` | `task_name` (each new Ad Set named after its task) or `custom` |
| `meta_ad_set_number_start` | first number when several new Ad Sets share a custom name |
| `meta_ad_set_name` | name of the new Ad Set |
| `meta_pixel` | Pixel name or id |
| `meta_daily_budget` | daily budget, e.g. `150` |
| `meta_countries` | ISO codes, e.g. `["IT","US"]` |
| `meta_audience` | `new_customers`, `existing_customers` |
| `meta_age_min`, `meta_age_max` | 18–65 |
| `meta_attribution` | `7d_click_1d_view`, `7d_click`, `1d_click`, `1d_view`, `7d_click_1d_view_1d_engaged` |
| `meta_bid_strategy` | `max_conversions`, `max_conversion_value` |
| `meta_start_time`, `meta_end_time` | `YYYY-MM-DDTHH:mm` in the ad account's time zone (the end must be in the future) |

## Google (`google_…`)

| Key | Value |
|---|---|
| `google_template` | Launch Template name or id, or `none` |
| `google_ad_account` | customer id or name |
| `google_creative` | `field_mapping` or a Creative Template |
| `google_destination` | `existing_ad_group`, `new_ad_group` (in an existing campaign), `new_campaign` |
| `google_campaign`, `google_campaign_name` | campaign name or id / name of a new campaign |
| `google_ad_group`, `google_ad_group_grouping` (`shared` / `separate`), `google_ad_group_name` | Ad Group choices |
| `google_daily_budget` | daily budget of a new campaign |
| `google_bidding` | `maximize_conversions`, `maximize_conversion_value`, `maximize_clicks` |
| `google_customer_type` | `new_customers`, `existing_customers` |
| `google_countries` | ISO codes |
| `google_start_time`, `google_end_time` | `YYYY-MM-DDTHH:mm` |

## AppLovin (`applovin_…`)

| Key | Value |
|---|---|
| `applovin_template`, `applovin_creative` | as for Meta |
| `applovin_destination` | `existing_campaign`, `new_campaign` |
| `applovin_campaign`, `applovin_campaign_name` | campaign name or id / name of a new campaign |
| `applovin_creative_set_name`, `applovin_creative_set_grouping` (`separate` / `shared`) | Creative Set choices |
| `applovin_daily_budget` | daily budget of a new campaign |
| `applovin_goal_type` | `chk_roas`, `cpp`, `cpe` |
| `applovin_goal_value`, `applovin_event_target`, `applovin_roas_day_target` (`day0` / `day7`) | goal details |
| `applovin_countries`, `applovin_start_time` | as above |
