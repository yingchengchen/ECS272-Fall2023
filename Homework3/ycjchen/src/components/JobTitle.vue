<script lang="ts">
import * as d3 from "d3";
import Data from '../../data/ds_salaries.json';

import axios from 'axios';
import { isEmpty, debounce } from 'lodash';

import { Dot, ComponentSize, Margin } from '../types';
// A "extends" B means A inherits the properties and methods from B.
interface Title extends Dot{
    salary_in_usd: number;
    job_title: string;
    cat_salary:string;
    company_location:string;
}


export default {
    inject: ['eventBusSalary'],
    data() {
        return {
            newdataC: [] as Title[],
            size: { width: 0, height: 0 } as ComponentSize,
            margin: {left: 40, right: 40, top: 0, bottom: 5} as Margin,
            salaryandcountry:'the salary range you select in your interested country',
        }
    },
    computed: {
        rerender() {
            return (!isEmpty(this.newdataC)) && this.size
        },
        update_country(){
            this.eventBusSalary.on('salarymsg',(data)=>{this.salaryandcountry = data})
            
            
        },
        

    
    },
    created() {
        if (isEmpty(Data)) return;
        this.eventBusSalary.on('salarymsg',(data)=>{this.salaryandcountry = data})
        const salaryCategories = [
            { range: [0, 50000], category: "0-50k" },
            { range: [50000, 100000], category: "50-100k" },
            { range: [100000, 150000], category: "100-150k" },
            { range: [150000, 200000], category: "150-200k" },
            { range: [200000, 250000], category: "200-250k" },
            { range: [250000, 300000], category: "250-300k" },
            { range: [300000, 450000], category: ">300k" },
            ];
        // this.newdataC = Data.data
        const newData: Title[] = Data.data.map((item) => {
            const salary = item.salary_in_usd;
            const category = salaryCategories.find((cat) => salary >= cat.range[0] && salary < cat.range[1]);
            return {
                ...item,
                cat_salary: category ? category.category : "Unknown",
            };
        })
        this.newdataC = newData
    },

    
    methods: {
        onResize() {  
            let target = this.$refs.donutContainer as HTMLElement
            if (target === undefined) return;
            this.size = { width: target.clientWidth, height: target.clientHeight };
        },
        
        initChart() {
            d3.select('#donut-svg').selectAll('*').remove()
            
            let donutContainer = d3.select('#donut-svg')
             
            let yTitle: string[] = [ ...new Set(this.newdataC.map((d: Title) => d.job_title as string))]
            let count = {};
            this.newdataC.map((d: Title) => count[d.job_title] = 1 + (count[d.job_title] || 0));
            const values = Object.values(count);

            // Sort the values in ascending order
            values.sort((a, b) => a - b);

            // Calculate the median
            const median = values[Math.floor(values.length / 2)];
            const sum = values.reduce((acc, value) => acc + value, 0);
            const average = sum / values.length;

            // Filter and accumulate values less than the median
            const lessThanMedian = values.filter(value => value < average);
            const sumOfLessThanMedian = lessThanMedian.reduce((acc, value) => acc + value, 0);

            // Create a new object with the "others" property
            const resultObject = { ...count, "Others": sumOfLessThanMedian };
            
            const newObject = {};
            for (const key in resultObject) {
                if (resultObject[key] >= average) {
                    newObject[key] = resultObject[key];
                }
            }
            const keysArray = Object.keys(newObject);            
            
            // Compute the position of each group on the pie:
            var pie = d3.pie()
                        .sort(null) // Do not sort group by size
                        .value(function(d) {return d[1]; })
            var data_ready = pie(Object.entries(newObject))
            var radius = Math.min(this.size.width, this.size.height) / 2
            // The arc generator
            var arc = d3.arc()
            .innerRadius(radius * 0.5)
            .outerRadius(radius * 0.8)

            // Another arc that won't be drawn. Just for labels positioning
            var outerArc = d3.arc()
            .innerRadius(radius * 0.9)
            .outerRadius(radius * 0.9)

            let donut = donutContainer.append('g')
            .attr('transform', 'translate(' + this.size.width/2 +  ',' + this.size.height/2 +')')
            .selectAll('allSlices')
            .data(data_ready)
            .enter()
            .append('path')
            .attr('d', arc)
            .attr('fill', function(d){ return("#E6E3DB"); }) //color(d.data[0])
            .attr("stroke", "white")
            .style("stroke-width", "2px")
            .style("opacity", 0.9)
            .transition()
            .duration(1000)
            .attrTween("d", function (d) {
                var i = d3.interpolate(d.endAngle, d.startAngle);
                return function (t) {
                    d.startAngle = i(t);
                    return arc(d);
                }
            });

            let polylines = donutContainer.append('g')
            .attr('transform', 'translate(' + this.size.width/2 +  ',' + this.size.height/2 +')')
            .selectAll('allPolylines')
            .data(data_ready)
            .enter()
            .append('polyline')
                .attr("stroke", "black")
                .style("fill", "none")
                .attr("stroke-width", 1)
                .attr('points', function(d) {
                var posA = arc.centroid(d) // line insertion in the slice
                var posB = outerArc.centroid(d) // line break: we use the other arc generator that has been built only for that
                var posC = outerArc.centroid(d); // Label position = almost the same as posB
                var midangle = d.startAngle + (d.endAngle - d.startAngle) / 2 // we need the angle to see if the X position will be at the extreme right or extreme left
                posC[0] = radius * 0.95 * (midangle < Math.PI ? 1:-1); // multiply by 1 or -1 to put it on the right or on the left
                return [posA, posB, posC]
                })

            let labels = donutContainer.append('g')
            .attr('transform', 'translate(' + this.size.width/2 +  ',' + this.size.height/2 +')')
            .selectAll('allLabels')
            .data(data_ready)
            .enter()
            .append('text')
                .text( function(d) { return d.data[0]+': '+d.data[1] } )
                .attr('transform', function(d) {
                    var pos = outerArc.centroid(d);
                    var midangle = d.startAngle + (d.endAngle - d.startAngle) / 2
                    pos[0] = radius * 0.99 * (midangle < Math.PI ? 1 : -1);
                    return 'translate(' + pos  + ')';
                })
                .style('text-anchor', function(d) {
                    var midangle = d.startAngle + (d.endAngle - d.startAngle) / 2
                    return (midangle < Math.PI ? 'start' : 'end')
                })
            .style('font-size', '1rem')
            
            // let title = donutContainer
            //             .append("g")
            //             .append("text")
            //             .attr('transform', `translate(${this.size.width / 2}, ${this.size.height})`)
            //             .style('text-anchor', 'middle')
            //             .style('font-weight', 'bold')
            //             .style('font-size', '1rem')
            //             .text("Fig. 3 Composition of Job Title of the Data Scientists for"+this.salaryandcountry)
            
        },
        no_chosen(){
            const box = d3.select('#donut-svg').append('g')
                .append("rect")
                .attr("x", this.margin.top+10 )
                .attr("y", this.margin.top)
                .attr("width", this.size.width-this.margin.right+10)
                .attr("height", this.size.height-this.margin.bottom+10 )
                .style("fill", " #E6E3DB")
                
            d3.select('#donut-svg').append('g')
            .append("text")
            .attr("x", 10) // X-coordinate of the text
            .attr("y", 80) // Y-coordinate of the first line
            // .style('font-weight', 'bold')
            .style('font-size', '1.2rem')
            .style("line-height", "1.2") // Line height for spacing
            .selectAll("tspan") // Create multiple <tspan> elements for each line
            .data([
            'From the the Sankey chart shown in Fig. 2, You may want to know the composition of the job',
            "titles for a particular salary range. Select the salary range you are interested in from the left",
            "axis of the Sankey chart.You will see a donut chart pops up HERE. It shows the composition",
            "of the job titles for the particular salary range you select. "
            ])
            .enter()
            .append("tspan")
            .text((d) => d)
            .attr("x", this.margin.left+20) // X-coordinate for each line
            .attr("dy", "1.5em"); // Adjust the vertical spacing between lines
            


        },
    },
    watch: {
        rerender(newSize) {
            if (!isEmpty(newSize)) {
                d3.select('#donut-svg').selectAll('*').remove() 
                this.no_chosen()
            }
        },
        salaryandcountry:function(value){
            
            const [newSalary,newCountry] = value.split(' (USD) in ');
            // this.country = newCountry
            // this.salary = newSalary
            const salaryCategories = [
                { range: [0, 50000], category: "0-50k" },
                { range: [50000, 100000], category: "50-100k" },
                { range: [100000, 150000], category: "100-150k" },
                { range: [150000, 200000], category: "150-200k" },
                { range: [200000, 250000], category: "200-250k" },
                { range: [250000, 300000], category: "250-300k" },
                { range: [300000, 450000], category: ">300k" },
            ];
            
            const newData: Title[] = Data.data.map((item) => {
                const salary = item.salary_in_usd;
                const category = salaryCategories.find((cat) => salary >= cat.range[0] && salary < cat.range[1]);
                return {
                    ...item,
                    cat_salary: category ? category.category : "Unknown",
                };
            })
            var isoCountries = {
                'AF' : 'Afghanistan',
                'AX' : 'Aland Islands',
                'AL' : 'Albania',
                'DZ' : 'Algeria',
                'AS' : 'American Samoa',
                'AD' : 'Andorra',
                'AO' : 'Angola',
                'AI' : 'Anguilla',
                'AQ' : 'Antarctica',
                'AG' : 'Antigua And Barbuda',
                'AR' : 'Argentina',
                'AM' : 'Armenia',
                'AW' : 'Aruba',
                'AU' : 'Australia',
                'AT' : 'Austria',
                'AZ' : 'Azerbaijan',
                'BS' : 'Bahamas',
                'BH' : 'Bahrain',
                'BD' : 'Bangladesh',
                'BB' : 'Barbados',
                'BY' : 'Belarus',
                'BE' : 'Belgium',
                'BZ' : 'Belize',
                'BJ' : 'Benin',
                'BM' : 'Bermuda',
                'BT' : 'Bhutan',
                'BO' : 'Bolivia',
                'BA' : 'Bosnia And Herzegovina',
                'BW' : 'Botswana',
                'BV' : 'Bouvet Island',
                'BR' : 'Brazil',
                'IO' : 'British Indian Ocean Territory',
                'BN' : 'Brunei Darussalam',
                'BG' : 'Bulgaria',
                'BF' : 'Burkina Faso',
                'BI' : 'Burundi',
                'KH' : 'Cambodia',
                'CM' : 'Cameroon',
                'CA' : 'Canada',
                'CV' : 'Cape Verde',
                'KY' : 'Cayman Islands',
                'CF' : 'Central African Republic',
                'TD' : 'Chad',
                'CL' : 'Chile',
                'CN' : 'China',
                'CX' : 'Christmas Island',
                'CC' : 'Cocos (Keeling) Islands',
                'CO' : 'Colombia',
                'KM' : 'Comoros',
                'CG' : 'Congo',
                'CD' : 'Congo, Democratic Republic',
                'CK' : 'Cook Islands',
                'CR' : 'Costa Rica',
                'CI' : 'Cote D\'Ivoire',
                'HR' : 'Croatia',
                'CU' : 'Cuba',
                'CY' : 'Cyprus',
                'CZ' : 'Czech Republic',
                'DK' : 'Denmark',
                'DJ' : 'Djibouti',
                'DM' : 'Dominica',
                'DO' : 'Dominican Republic',
                'EC' : 'Ecuador',
                'EG' : 'Egypt',
                'SV' : 'El Salvador',
                'GQ' : 'Equatorial Guinea',
                'ER' : 'Eritrea',
                'EE' : 'Estonia',
                'ET' : 'Ethiopia',
                'FK' : 'Falkland Islands (Malvinas)',
                'FO' : 'Faroe Islands',
                'FJ' : 'Fiji',
                'FI' : 'Finland',
                'FR' : 'France',
                'GF' : 'French Guiana',
                'PF' : 'French Polynesia',
                'TF' : 'French Southern Territories',
                'GA' : 'Gabon',
                'GM' : 'Gambia',
                'GE' : 'Georgia',
                'DE' : 'Germany',
                'GH' : 'Ghana',
                'GI' : 'Gibraltar',
                'GR' : 'Greece',
                'GL' : 'Greenland',
                'GD' : 'Grenada',
                'GP' : 'Guadeloupe',
                'GU' : 'Guam',
                'GT' : 'Guatemala',
                'GG' : 'Guernsey',
                'GN' : 'Guinea',
                'GW' : 'Guinea-Bissau',
                'GY' : 'Guyana',
                'HT' : 'Haiti',
                'HM' : 'Heard Island & Mcdonald Islands',
                'VA' : 'Holy See (Vatican City State)',
                'HN' : 'Honduras',
                'HK' : 'Hong Kong',
                'HU' : 'Hungary',
                'IS' : 'Iceland',
                'IN' : 'India',
                'ID' : 'Indonesia',
                'IR' : 'Iran, Islamic Republic Of',
                'IQ' : 'Iraq',
                'IE' : 'Ireland',
                'IM' : 'Isle Of Man',
                'IL' : 'Israel',
                'IT' : 'Italy',
                'JM' : 'Jamaica',
                'JP' : 'Japan',
                'JE' : 'Jersey',
                'JO' : 'Jordan',
                'KZ' : 'Kazakhstan',
                'KE' : 'Kenya',
                'KI' : 'Kiribati',
                'KR' : 'Korea',
                'KW' : 'Kuwait',
                'KG' : 'Kyrgyzstan',
                'LA' : 'Lao People\'s Democratic Republic',
                'LV' : 'Latvia',
                'LB' : 'Lebanon',
                'LS' : 'Lesotho',
                'LR' : 'Liberia',
                'LY' : 'Libyan Arab Jamahiriya',
                'LI' : 'Liechtenstein',
                'LT' : 'Lithuania',
                'LU' : 'Luxembourg',
                'MO' : 'Macao',
                'MK' : 'Macedonia',
                'MG' : 'Madagascar',
                'MW' : 'Malawi',
                'MY' : 'Malaysia',
                'MV' : 'Maldives',
                'ML' : 'Mali',
                'MT' : 'Malta',
                'MH' : 'Marshall Islands',
                'MQ' : 'Martinique',
                'MR' : 'Mauritania',
                'MU' : 'Mauritius',
                'YT' : 'Mayotte',
                'MX' : 'Mexico',
                'FM' : 'Micronesia, Federated States Of',
                'MD' : 'Moldova',
                'MC' : 'Monaco',
                'MN' : 'Mongolia',
                'ME' : 'Montenegro',
                'MS' : 'Montserrat',
                'MA' : 'Morocco',
                'MZ' : 'Mozambique',
                'MM' : 'Myanmar',
                'NA' : 'Namibia',
                'NR' : 'Nauru',
                'NP' : 'Nepal',
                'NL' : 'Netherlands',
                'AN' : 'Netherlands Antilles',
                'NC' : 'New Caledonia',
                'NZ' : 'New Zealand',
                'NI' : 'Nicaragua',
                'NE' : 'Niger',
                'NG' : 'Nigeria',
                'NU' : 'Niue',
                'NF' : 'Norfolk Island',
                'MP' : 'Northern Mariana Islands',
                'NO' : 'Norway',
                'OM' : 'Oman',
                'PK' : 'Pakistan',
                'PW' : 'Palau',
                'PS' : 'Palestinian Territory, Occupied',
                'PA' : 'Panama',
                'PG' : 'Papua New Guinea',
                'PY' : 'Paraguay',
                'PE' : 'Peru',
                'PH' : 'Philippines',
                'PN' : 'Pitcairn',
                'PL' : 'Poland',
                'PT' : 'Portugal',
                'PR' : 'Puerto Rico',
                'QA' : 'Qatar',
                'RE' : 'Reunion',
                'RO' : 'Romania',
                'RU' : 'Russian Federation',
                'RW' : 'Rwanda',
                'BL' : 'Saint Barthelemy',
                'SH' : 'Saint Helena',
                'KN' : 'Saint Kitts And Nevis',
                'LC' : 'Saint Lucia',
                'MF' : 'Saint Martin',
                'PM' : 'Saint Pierre And Miquelon',
                'VC' : 'Saint Vincent And Grenadines',
                'WS' : 'Samoa',
                'SM' : 'San Marino',
                'ST' : 'Sao Tome And Principe',
                'SA' : 'Saudi Arabia',
                'SN' : 'Senegal',
                'RS' : 'Serbia',
                'SC' : 'Seychelles',
                'SL' : 'Sierra Leone',
                'SG' : 'Singapore',
                'SK' : 'Slovakia',
                'SI' : 'Slovenia',
                'SB' : 'Solomon Islands',
                'SO' : 'Somalia',
                'ZA' : 'South Africa',
                'GS' : 'South Georgia And Sandwich Isl.',
                'ES' : 'Spain',
                'LK' : 'Sri Lanka',
                'SD' : 'Sudan',
                'SR' : 'Suriname',
                'SJ' : 'Svalbard And Jan Mayen',
                'SZ' : 'Swaziland',
                'SE' : 'Sweden',
                'CH' : 'Switzerland',
                'SY' : 'Syrian Arab Republic',
                'TW' : 'Taiwan',
                'TJ' : 'Tajikistan',
                'TZ' : 'Tanzania',
                'TH' : 'Thailand',
                'TL' : 'Timor-Leste',
                'TG' : 'Togo',
                'TK' : 'Tokelau',
                'TO' : 'Tonga',
                'TT' : 'Trinidad And Tobago',
                'TN' : 'Tunisia',
                'TR' : 'Turkey',
                'TM' : 'Turkmenistan',
                'TC' : 'Turks And Caicos Islands',
                'TV' : 'Tuvalu',
                'UG' : 'Uganda',
                'UA' : 'Ukraine',
                'AE' : 'United Arab Emirates',
                'GB' : 'United Kingdom',
                'US' : 'United States',
                'UM' : 'United States Outlying Islands',
                'UY' : 'Uruguay',
                'UZ' : 'Uzbekistan',
                'VU' : 'Vanuatu',
                'VE' : 'Venezuela',
                'VN' : 'Viet Nam',
                'VG' : 'Virgin Islands, British',
                'VI' : 'Virgin Islands, U.S.',
                'WF' : 'Wallis And Futuna',
                'EH' : 'Western Sahara',
                'YE' : 'Yemen',
                'ZM' : 'Zambia',
                'ZW' : 'Zimbabwe'
            };
            function getCountryName (countryCode) {
                    if (isoCountries.hasOwnProperty(countryCode)) {
                        return isoCountries[countryCode];
                    } else {
                        return countryCode;
                    }
            }
            this.newdataC = newData.filter((d) => (d.cat_salary === newSalary)&&(getCountryName(d.company_location) === newCountry))
            this.initChart()
                
        }
        
        
            
        
        
    },
    
    mounted() {
        window.addEventListener('resize', debounce(this.onResize, 100)) 
        this.onResize()
    },
    beforeDestroy() {
       window.removeEventListener('resize', this.onResize)
    }
}

</script>

<template>
    <div class="chart-container-donut" ref="donutContainer">
        <svg id="donut-svg" width="100%" height="100%" >
        </svg>
        <div class="titlebox" >
            <p class="title">Fig. 3 Composition of Job Title of the Data Scientists for {{ salaryandcountry }} </p> 
        </div>
    </div>
</template>

<style >
.chart-container-donut{
    height: 100%;
}
/* .title{
    
    font-weight: bold;
    font-size: 1rem;
    text-align: center;
} */

</style>

