<div class="question-block">

    <span class="question-label">
        Do you have current pain?
        (None must be unchecked if you wish to select anything else)
    </span>

    <div class="option-list option-list-grid">

        @for (
            option of painLocationOptions;
            track option
        ) {

            <label
                class="radio-option"
                [class.disabled-choice]="
                    isPainDisabled(option)
                "
            >

                <input
                    type="checkbox"
                    [checked]="
                        isPainSelected(option)
                    "
                    [disabled]="
                        isPainDisabled(option)
                    "
                    (change)="
                        togglePainLocation(option)
                    "
                />

                {{ painLocationLabel(option) }}

            </label>

        }

    </div>

</div>









readonly painLocationOptions = [
    'None',
    'Back',
    'R Hip',
    'L Hip',
    'R Knee',
    'L Knee',
    'R Ankle',
    'L Ankle',
    'R Foot',
    'L Foot',
    'R Thigh',
    'L Thigh',
    'R Lower leg',
    'L Lower leg'
];

painLocationLabel(location: string): string {
    if (location.startsWith('R ')) {
        return `Right ${location.substring(2)}`;
    }

    if (location.startsWith('L ')) {
        return `Left ${location.substring(2)}`;
    }

    return location;
}

isPainSelected(location: string): boolean {
    return this.answers.painLocations.includes(location);
}

isPainDisabled(location: string): boolean {
    return (
        location !== 'None' &&
        this.isPainSelected('None')
    );
}

togglePainLocation(location: string): void {
    const selected =
        this.isPainSelected(location);

    if (location === 'None') {

        if (selected) {
            this.answers.painLocations = [];
        } else {
            this.answers.painLocations = ['None'];
        }

        return;
    }

    if (this.isPainSelected('None')) {
        return;
    }

    if (selected) {
        this.answers.painLocations =
            this.answers.painLocations.filter(
                item => item !== location
            );
    } else {
        this.answers.painLocations = [
            ...this.answers.painLocations,
            location
        ];
    }
}
