<script lang="ts">
import * as d3 from "d3";
import { sankey as d3Sankey, sankeyLinkHorizontal as d3SsankeyLinkHorizontal } from 'd3-sankey';
import Data from '../../data/ds_salaries.json';
import app from '../main.js';
// import Data from '../../data/test.json';

import axios from 'axios';
import { isEmpty, debounce } from 'lodash';
import {ref, watch} from 'vue';
import { Dot, ComponentSize, Margin } from '../types';
// A "extends" B means A inherits the properties and methods from B.

// let country = ref('US');
interface JobInfo extends Dot{
    experience_level: string;
    salary_in_usd: number;
    company_size:string;
    remote_ratio:string;
    cat_salary:string;
}

export default {
    inject: ['eventBus','eventBusSalary'],
    data() {
        // Here we define the local states of this component. If you think the component as a class, then these are like its private variables.
        return {
            newdata: [] as JobInfo[],
            size: { width: 0, height: 0 } as ComponentSize,
            margin: {left: 50, right: 50, top: 10, bottom: 30} as Margin,
            country: 'your interested country',
        }
    },
    computed: {
        // Re-render the chart whenever the window is resized or the data changes (and data is non-empty)
        rerender() {
            return (!isEmpty(this.newdata)) && this.size
        },
        update_country(){
            this.eventBus.on('countrymsg',(data)=>{this.country = data})
            
            
        },
        

    
    },
    // Anything in here will only be executed once.
    // Refer to the lifecycle in Vue.js for more details, mentioned at the very top of this file.
    created() {
        if (isEmpty(Data)) return;
        this.eventBus.on('countrymsg',(data)=>{this.country = data; })
        this.newdata = Data.data
        
    },
    methods: {
        onResize() {  // record the updated size of the target element
            let target = this.$refs.lineContainer as HTMLElement
            if (target === undefined) return;
            this.size = { width: target.clientWidth, height: target.clientHeight };
        },
        initSan(){
            d3.select('#line-svg').selectAll('*').remove()
            let lineContainer = d3.select('#line-svg').append("g")
                                    .call(d3.zoom().on("zoom", function (e) {
                                        lineContainer.attr("transform", e.transform)
                                    }))
            // Define the salary categories and their corresponding ranges
            const salaryCategories = [
            { range: [0, 50000], category: "0-50k" },
            { range: [50000, 100000], category: "50-100k" },
            { range: [100000, 150000], category: "100-150k" },
            { range: [150000, 200000], category: "150-200k" },
            { range: [200000, 250000], category: "200-250k" },
            { range: [250000, 300000], category: "250-300k" },
            { range: [300000, 450000], category: ">300k" },
            ];
            
            this.newdata.forEach((item) => {
                const salary = item.salary_in_usd;
                const category = salaryCategories.find((cat) => salary >= cat.range[0] && salary < cat.range[1]);
                item.cat_salary = category ? category.category : "Unknown";
            });
            const updatedData = this.newdata.map((d) => ({
                    ...d,
                    remote_ratio: d.remote_ratio.toString()
            }));
            // Define possible values for each property
            const experienceLevels = ["EX","SE","MI","EN"];
            const catSalaries = [">300k","250-300k", "200-250k","150-200k","100-150k","50-100k","0-50k"]; 
            const remoteRatios = ["100", "50", "0"];
            const companySizes = ["L","M","S"];

            // Iterate over the combinations and count occurrences
            // console.log(this.newdata);
            const result=[]
            catSalaries.forEach((catSalary) => {
                experienceLevels.forEach((experienceLevel) => {
                    companySizes.forEach((companySize) => {
                        const count = updatedData.filter((item) =>
                            item.cat_salary === catSalary &&
                            item.experience_level === experienceLevel &&
                            item.company_size === companySize
                        ).length;
                        // console.log(count)

                    result.push({
                        
                        "cat_salary": catSalary,
                        "experience_level": experienceLevel,
                        "company_size": companySize,
                        "value": count,
                    });
                    });
                });
            });

            // console.log(result);
            
            const keys = ["cat_salary","experience_level","company_size"]
            let index = -1;
            const Nodes = [];
            const nodeByKey = new d3.InternMap([], JSON.stringify);;
            const indexByKey = new d3.InternMap([], JSON.stringify);;
            const Links = [];

            for (const k of keys) {
                for (const d of result) {
                const key = [k, d[k]];
                if (nodeByKey.has(key)) continue;
                const node = {name: d[k]};
                Nodes.push(node);
                nodeByKey.set(key, node);
                indexByKey.set(key, ++index);
                }
            }

            for (let i = 1; i < keys.length; ++i) {
                const a = keys[i - 1];
                const b = keys[i];
                const prefix = keys.slice(0, i + 1);
                const linkByKey = new d3.InternMap([], JSON.stringify);
                for (const d of result) {
                const names = prefix.map(k => d[k]);
                const value = d.value || 0;
                let link = linkByKey.get(names);
                if (link) { link.value += value; continue; }
                link = {
                    source: indexByKey.get([a, d[a]]),
                    target: indexByKey.get([b, d[b]]),
                    names,
                    value
                };
                Links.push(link);
                linkByKey.set(names, link);
                }
            }
            
            const sankey = d3Sankey()
                            .nodeSort(null)
                            .linkSort(null)
                            .nodeWidth(5)
                            .nodePadding(15)
                            .extent([[0, 30], [this.size.width-this.margin.right, this.size.height-this.margin.bottom]])

            const {nodes, links} = sankey({
                nodes: Nodes.map(d => Object.create(d)),
                links: Links.map(d => Object.create(d))
            });
            // console.log({nodes, links})
            let axislabel1 = lineContainer.append("g")
                            .append('text')
                            .attr("transform", d => `translate(${this.margin.left-10},${this.margin.top+30})`)
                            .style('text-anchor', 'middle')
                            .style('font-size', '1rem')
                            .style('font-weight','bold')
                            .text('salary (USD)') // text content
            let axislabel2 = lineContainer.append("g")
                            .append('text')
                            .attr("transform", d => `translate(${this.size.width / 2},${this.margin.top+30})`)
                            .style('text-anchor', 'middle')
                            .style('font-size', '1rem')
                            .style('font-weight','bold')
                            .text('experience level') // text content
            let axislabel3 = lineContainer.append("g")
                            .append('text')
                            .attr("transform", d => `translate(${this.size.width-this.margin.right},${this.margin.top+30})`)
                            .style('text-anchor', 'middle')
                            .style('font-size', '1rem')
                            .style('font-weight','bold')
                            .text('company size') // text content
            const color = d3.scaleOrdinal(d3.schemeGnBu[7]).domain(["0-50k","50-100k","100-150k","150-200k","200-250k","250-300k",">300k"])
            // const color = d3.scaleSequential(d3.interpolate("white","black")).domain([1,8])
            // console.log(links)
            let alllinks = lineContainer.append("g")
                        .attr("transform", d => `translate(10,20)`)
                        .attr("fill", "none")
                        .selectAll("g")
                        .data(links)
                        .join("path")
                        .attr("class","links")
                        .attr("d", d3SsankeyLinkHorizontal())
                        .attr("stroke",d => color(d.names[0]))
                        .attr("stroke-width", d => d.width)
                        .style("mix-blend-mode", "multiply")
                        .append("title")
                        .text(d => `${d.names.join(" → ")}\n${d.value.toLocaleString()}`);
            
            let allnodes = lineContainer.append("g")
                        .attr("transform", d => `translate(10,20)`)
                        .selectAll("rect")
                        .data(nodes)
                        .join("rect")
                        .attr("class",function(d){
                                                    if(!experienceLevels.includes(d.name) && !companySizes.includes(d.name)){
                                                        return "nodes"
                                                    }
                                                    else{
                                                        return "othernodes"
                                                 }})
                        .attr("x", d => d.x0)
                        .attr("y", d => d.y0)
                        .attr("height", d => d.y1 - d.y0)
                        .attr("width", d => d.x1 - d.x0 +2)
                        .attr("fill","#2a2727")
                        .append("title")
                        .text(d => `${d.name}\n${d.value.toLocaleString()}`);
            let self = this
            const selectionSalary = d3.selectAll("rect.nodes")
                                        .on('mouseover',function(e,d){
                                            d3.select(this)
                                            .style("fill", "indianred")
                                            // const str = `${d.name} (USD) in ${self.country}`
                                            // self.eventBusSalary.emit('salarymsg',str)})
                                        .on('mouseout',function(){
                                            d3.select(this)
                                            .style("fill", "#2a2727");
                                        })
                                        .on('click', function(e,d){
                                            const str = `${d.name} (USD) in ${self.country}`
                                            self.eventBusSalary.emit('salarymsg',str)})
                                        })
                                    
                                    
                                      
            
            
            let labels = lineContainer.append("g")
                        .attr("transform", d => `translate(10,20)`)
                        .style("font", "10px sans-serif")
                        .selectAll("text")
                        .data(nodes)
                        .join("text")
                        .attr("x", d => d.x0 < this.size.width / 2 ? d.x1 + 6 : d.x0 - 6)
                        .attr("y", d => (d.y1 + d.y0) / 2)
                        .attr("dy", "0.35em")
                        .attr("text-anchor", d => d.x0 < this.size.width / 2 ? "start" : "end")
                        .text(d => d.name)
            let legendElementWidth = 50
            let legendHeight = 10         
            let legends = lineContainer.append("g")
                        .attr("transform", d => `translate(10,-${this.size.height*0.92})` )
                        .selectAll("rect")
                        .data(catSalaries.reverse())
                        .enter()
                        .append("rect")
                        .attr("x", (d, i) => legendElementWidth * i)
                        .attr("y", this.size.height - (2*legendHeight))
                        .attr("width", legendElementWidth)
                        .attr("height", legendHeight)
                        .style("fill", d => color(d));
            lineContainer.append("g")
                        .attr("transform", d => `translate(10,-${this.size.height*0.91})` )
                        .selectAll("text")
                        .data(catSalaries)
                        .enter()
                        .append("text")
                        .text(d => d )
                        .attr("x", (d, i) => legendElementWidth * i)
                        .attr("y", this.size.height - (legendHeight / 2)+5)
                        .style("font-size", "1rem")
                        .style("fill", "#A4A4A4")
                        .style('font-weight', 'bold')
                        
        },
        no_chosen(){
            const box = d3.select('#line-svg').append('g')
                .append("rect")
                .attr("x", this.margin.top )
                .attr("y", this.margin.top)
                .attr("width", this.size.width-this.margin.right+20)
                .attr("height", this.size.height-this.margin.bottom+20 )
                .style("fill", " #E6E3DB")

            
            d3.select('#line-svg').append('g')
                .append("text")
                .attr("x", 20) // X-coordinate of the text
                .attr("y", 120) // Y-coordinate of the first line
                // .style('font-weight', 'bold')
                .style('font-size', '1.2rem')
                .style("line-height", "1.2") // Line height for spacing
                .selectAll("tspan") // Create multiple <tspan> elements for each line
                .data([
                'Select the country you are interested in from the y-axis. HERE will pop up a Sankey chart',
                "showing the correspondence between salary range, experience level and company size for",
                "the data scientists in the country you select. Hover your mouse over the nodes and links in",
                "Fig. 2, the detailed information of the correspondence under the mouse will pop up. "
                ])
                .enter()
                .append("tspan")
                .text((d) => d)
                .attr("x", this.margin.left+10) // X-coordinate for each line
                .attr("dy", "1.5em"); // Adjust the vertical spacing between lines
             

        },

    },
    watch: {
        
        rerender(newSize) {
            if (!isEmpty(newSize)) {
                d3.select('#line-svg').selectAll('*').remove() 
                this.no_chosen()
            }
        },
        country: function(value){
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
            this.newdata = Data.data.filter((d) => (getCountryName(d.company_location) == value))
                    
            this.initSan()
        },
        
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
    <div class="chart-container-line" ref="lineContainer">
        <svg id="line-svg" width="100%" height="100%" >
        </svg>
        <div class="titlebox" >
            <p class="title">Fig. 2 Data Scientist Job Salaries with corresponding experience level and company size in {{ country }}</p> 
            <!-- -->
        </div>
        
    </div>
</template>

<style >
.chart-container-line{
    height: 100%;
}

.title{
    font-weight: bold;
    font-size: 1rem;
    text-align: center;
}
</style>

