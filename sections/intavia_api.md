#### Techstack<!-- .element: style="float: left" -->

* FastAPI<!-- .element: class="fragment" -->
* including pydantic for type checking<!-- .element: class="fragment" -->
* Jinja2 (for SPARQL templates)<!-- .element: class="fragment" -->
* Blazegraph<!-- .element: class="fragment" -->
* custom library for conversion<!-- .element: class="fragment" -->

+++

#### API data flow<!-- .element: style="float: left; margin-top: 3%" -->


<div data-animate data-src="images/intavia_api_schema.drawio.svg">
<!--
{ "setup": [
{ "element": "#cell-7, #cell-11", "modifier": "attr", "parameters": [ {"class": "fragment", "data-fragment-index": "0"} ]},
{ "element": "#cell-12, #cell-8", "modifier": "attr", "parameters": [ {"class": "fragment", "data-fragment-index": "1"} ]},
{ "element": "#cell-13, #cell-9", "modifier": "attr", "parameters": [ {"class": "fragment", "data-fragment-index": "2"} ]},
{ "element": "#cell-14, #cell-10", "modifier": "attr", "parameters": [ {"class": "fragment", "data-fragment-index": "3"} ]},
{ "element": "#cell-15, #cell-19", "modifier": "attr", "parameters": [ {"class": "fragment", "data-fragment-index": "4"} ]}
]}
-->
</div>

+++

#### SPARQL template example I

<pre><code data-trim data-noescape data-line-numbers="12,23|15|21">
    PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
    PREFIX idm: <https://www.intavia.eu/idm/>
    PREFIX idmcore: <http://www.intavia.eu/idm-core/>
    PREFIX bioc: <http://ldf.fi/schema/bioc/>
    PREFIX owl: <http://www.w3.org/2002/07/owl#>

    SELECT ?entity ?entityType ?entityTypeLabel ?entityLabel ?gender ?genderLabel ?nationalityLabel ?occupation ?occupationLabel 
    ?event ?linkedIds ?count ?geometry ?role_type (?entity as ?source) ?mediaObject ?biographyObject

    {% include 'add_datasets_v2_1.sparql' %}

    WITH {
    SELECT DISTINCT ?entity ?entityTypeLabel 

    {% include 'add_datasets_v2_1.sparql' %}

    WHERE {
    {% include 'query_entities_v2_1.sparql' %}
    {% include 'entity_type_bindings_v2_1.sparql' %}
    } ORDER BY ?entity
    LIMIT {{limit}}
    {% if _offset > 0 %}OFFSET {{_offset}}{% endif %}
    } AS %query_set

    WITH {
        SELECT (COUNT(DISTINCT ?entity) AS ?count)

        {% for dataset in datasets %}
        FROM <{{dataset.value}}>
        {% endfor %}

        WHERE {
            {% include 'query_entities_v2_1.sparql' %}
            {% include 'entity_type_bindings_v2_1.sparql' %}
        }
    } AS %count_set

    WHERE {  
    INCLUDE %query_set
    INCLUDE %count_set
    {% include 'retrieve_entities_v2_1.sparql' %}
    }
</code></pre>

+++

#### SPARQL template example II


<pre><code data-trim data-noescape data-line-numbers="1-4|11-16|29-34">
{%- if q -%}
?entity (crm:P1_is_identified_by/rdfs:label)|rdfs:label ?entityLabel .
?entityLabel bds:search "{{q}}" .
{%- endif -%}
{%- if occupation -%}
    ?entity bioc:has_occupation ?occupation1 . 
    ?occupation1 rdfs:label ?occupationLabel1 .
    ?occupationLabel1 bds:search "{{occupation}}" .
    ?occupationLabel1 bds:matchAllTerms "true" .
{%- endif -%}
{%- if occupations_id -%}
    {%- for occ in occupations_id -%}
        {?entity bioc:has_occupation <{{occ}}>}
        {% if not loop.last %}UNION{% endif %}
        {%- endfor -%}
{%- endif -%}
{%- if born_before -%}
    ?bornBeforeEvent a crm:E67_Birth .
    ?bornBeforeEvent crm:P98_brought_into_life ?entity .
    ?bornBeforeEvent crm:P4_has_time-span/(crm:P82a_begin_of_the_begin|crm:P82b_end_of_the_end) ?bornBeforeFilter .
    FILTER(?bornBeforeFilter < xsd:dateTime("{{born_before}}"))
{%- endif -%}
{%- if born_after -%}
    ?bornAfterEvent a crm:E67_Birth .
    ?bornAfterEvent crm:P98_brought_into_life ?entity .
    ?bornAfterEvent crm:P4_has_time-span/(crm:P82a_begin_of_the_begin|crm:P82b_end_of_the_end) ?bornAfterFilter .
    FILTER(?bornAfterFilter > xsd:dateTime("{{born_after}}"))
{%- endif -%}
{%- if died_before -%}
    ?diedBeforeEvent a crm:E69_Death .
    ?diedBeforeEvent crm:P100_was_death_of ?entity .
    ?diedBeforeEvent crm:P4_has_time-span/(crm:P82a_begin_of_the_begin|crm:P82b_end_of_the_end) ?diedBeforeFilter .
    FILTER(?diedBeforeFilter < xsd:dateTime("{{died_before}}"))
{%- endif -%}
{%- if died_after -%}
    ?diedAfterEvent a crm:E69_Death .
    ?diedAfterEvent crm:P100_was_death_of ?entity .
    ?diedAfterEvent crm:P4_has_time-span/(crm:P82a_begin_of_the_begin|crm:P82b_end_of_the_end) ?diedAfterFilter .
    FILTER(?diedAfterFilter > xsd:dateTime("{{died_after}}"))
{%- endif -%}
{%- if kind -%}
    {%- for k1 in kind -%}
    {?entity a {{k1.get_rdf_uri()}}}
        {% if not loop.last %}UNION{% endif %}
    {%- endfor -%}
{%- endif -%}
{%- if gender -%}
    ?entity bioc:has_gender|bioc:gender bioc:{{gender.name}} .
{%- endif -%}
{%- if gender_id -%}
    ?entity bioc:has_gender|bioc:gender <{{gender_id}}> .
{%- endif -%}
{%- if related_entity -%}
?entity bioc:bearer_of ?role . ?role ^bioc:had_participant_in_role ?event .
    ?event (crm:P7_took_place_at|(bioc:had_participant_in_role/^bioc:bearer_of)) ?evEntity .  
        {?evEntity crm:P1_is_identified_by ?evEntityAppelation .
                ?evEntityAppelation a crm:E33_E41_Linguistic_Appellation .
                ?evEntityAppelation rdfs:label ?evEntityLabel } UNION {
                    ?evEntity rdfs:label ?evEntityLabel }
    ?evEntityLabel bds:search "{{related_entity}}" .
{%- endif -%}
{%- if related_entities_id -%}
?entity bioc:bearer_of ?role . ?role ^bioc:had_participant_in_role ?event .
    {%- for entity in related_entities_id -%}
        {?event (crm:P7_took_place_at|(bioc:had_participant_in_role/^bioc:bearer_of)) <{{entity}}>}
        {% if not loop.last %}UNION{% endif %}
        {%- endfor -%}
{%- endif -%}
{%- if event_role -%}
?entity bioc:bearer_of ?role .
    ?role a/rdfs:label ?roleLabel .
    ?roleLabel bds:search "{{event_role}}" .
    ?roleLabel bds:matchAllTerms "true" .
{%- endif -%}
{%- if event_roles_id -%}
?entity bioc:bearer_of ?role .
    {%- for role in event_roles_id -%}
        {?role a <{{role}}>}
        {% if not loop.last %}UNION{% endif %}
        {%- endfor -%}
{%- endif -%}
</code></pre>

+++

#### SPARQL template example III


<pre><code data-trim data-noescape data-line-numbers="1,17|2-4|13">
BIND("no label provided" AS ?defaultEntityLabel)
OPTIONAL {?entity crm:P1_is_identified_by ?appellation .
{?appellation a crm:E33_E41_Linguistic_Appellation .} UNION {?appellation a crm:E35_Title}
?appellation rdfs:label ?entityLabelPre .}
OPTIONAL {?entity bioc:has_occupation ?occupation . ?occupation rdfs:label ?occupationLabel .}
OPTIONAL {?entity owl:sameAs ?linkedIds}
OPTIONAL {?entity bioc:has_gender ?gender 
    OPTIONAL {?gender rdfs:label ?genderLabel }}
OPTIONAL {?entity bioc:has_nationality ?nationality . ?nationality rdfs:label ?nationalityLabel .}
OPTIONAL {?entity bioc:bearer_of ?role . 
        ?role ^bioc:had_participant_in_role|^bioc:occured_in_the_presence_of_in_role ?event .
        ?role a ?role_type
}
OPTIONAL {?entity crm:P168_place_is_defined_by/crm:P168_place_is_defined_by ?geometry}
OPTIONAL {?entity ^crm:P70_documents ?mediaObject }
OPTIONAL {?entity idmcore:bio_link ?biographyObject }
BIND(COALESCE(?entityLabelPre, ?defaultEntityLabel) AS ?entityLabel)
</code></pre>

+++

#### Models example


<pre><code data-trim data-noescape data-line-numbers="2-5|32-34">
class Entity(IntaViaBackendBaseModel):
    id: str = Field(
        ...,
        rdfconfig=FieldConfigurationRDF(path="entity", anchor=True, encode_function=pp_base64),
    )
    label: InternationalizedLabel | None = Field(
        None, rdfconfig=FieldConfigurationRDF(path="entityLabel", default_dict_key="default")
    )
    kind: EntityType = Field(EntityType.Person, rdfconfig=FieldConfigurationRDF(path="entityTypeLabel"))
    source: Source | None = Field(
        None, rdfconfig=FieldConfigurationRDF(callback_function=pp_source, path="source", default_dict_key="citation")
    )

    linkedIds: list[LinkedId] | None = Field(
        None, rdfconfig=FieldConfigurationRDF(callback_function=pp_id_provider, path="linkedIds", default_dict_key="id")
    )
    # _linkedIds: list[HttpUrl] | None = None
    gender: GenderType | None = Field(
        None, rdfconfig=FieldConfigurationRDF(callback_function=pp_gender_to_label, path="gender")
    )
    occupations: typing.List[Occupation] | None = None
    alternativeLabels: list[InternationalizedLabel] | None = Field(
        None, rdfconfig=FieldConfigurationRDF(path="entityLabel", default_dict_key="default")
    )
    description: str | None = None
    media: list[str] | None = Field(
        None, rdfconfig=FieldConfigurationRDF(path="mediaObject", encode_function=pp_base64_to_list)
    )
    biographies: list[str] | None = Field(
        None, rdfconfig=FieldConfigurationRDF(path="biographyObject", encode_function=pp_base64_to_list)
    )
    geometry: typing.Union[Polygon, Point] | None = Field(
        None, rdfconfig=FieldConfigurationRDF(path="geometry", callback_function=pp_lat_long, bypass_data_mapping=True)
    )
    relations: list[EntityEventRelation] = []
</code></pre>